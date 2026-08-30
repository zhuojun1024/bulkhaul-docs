# 生产化落地方案（bulkhaul → 真产品）

> 2026-08-30 基于全量代码扫描 + 架构评审制定。覆盖安全 / 数据架构 / 运维可观测 / 开发体验
> 四大类缺口，给出可执行任务、优先级、依赖、验收标准。
> 纪律：每项先实现 → 补测试/验证 → 全绿 → 才标 done-verified。
> 状态：pending / in-progress / done-verified / blocked
>
> 关联文档：backend-plan.md（后端架构）、api-integration-plan.md（接口联调）、
> fix-plan.md（整改进度）、flow-audit.md（业务闭环审计）、lessons-learned.md（验证/运维经验）。

---

## 0. 现状定位与目标

**现状**：打磨良好的演示 / POC。接口、鉴权（JWT+bcrypt+验证码）、服务端 RBAC 单点守卫、
审计日志、Flyway 迁移、真 MySQL+Redis、三层测试（141 集成 + 556 服务 + 82 UI）均已到位。

**目标**：真产品——真实客户、真实数据、多用户、水平可扩展、可部署、可观测、防回归。

**核心判断**：缺的不是"功能"，而是"生产那层皮" + 两个**架构级决定**（存储模型 B1、并发模型 B3）。
地基是实的；下面按"上线会被卡住"的优先级排，分 A 安全 / B 数据架构 / C 运维 / D 开发体验四类。

**已站得住（别低估）**：真 JWT+bcrypt+验证码、服务端 RBAC 操作级守卫（单点可信）、服务端审计、
Flyway、真 MySQL+Redis、三层测试、全局异常处理、数据权限**已建模**（dataScopes 集合 + setDataScope 端点，
只差读端点强制）。

---

## 1. 缺口总览（按优先级）

| 编号 | 类别 | 缺口 | 优先级 | 工作量 | 依赖 |
|---|---|---|---|---|---|
| A1 | 安全 | 行级数据权限服务端强制 | **P0** | 中 | B1（过滤实现方式） |
| A2 | 安全 | 登录防爆破 + 锁定（服务端） | **P0** | 小 | — |
| A4 | 安全 | 密钥/配置外置（生产 profile） | **P0** | 小 | — |
| C1 | 运维 | 部署工件（Dockerfile + compose） | **P0** | 中 | A4 |
| C2 | 运维 | 后端 CI | **P0** | 小 | — |
| B1 | 架构 | 存储模型决策（JSON blob → 规范化） | **P0** | 大 | — |
| B3 | 架构 | 并发/冲突模型（乐观锁 / 多实例） | **P0** | 中 | B1 |
| A3 | 安全 | 全局限流 | P1 | 小 | A2 |
| A5 | 安全 | 输入校验（@Valid DTO） | P1 | 中 | — |
| B2 | 架构 | 分页 | P1 | 中 | B1 |
| C3 | 运维 | 可观测性（actuator + 结构化日志） | P1 | 小 | C1 |
| C4 | 运维 | 服务端定时任务（leader 单实例） | P1 | 中 | B3 |
| C5 | 运维 | CORS / 部署拓扑（Nginx 反代） | P1 | 小 | C1 |
| D1 | 开发 | OpenAPI / Swagger | P1 | 小 | — |
| D2 | 开发 | API 契约测试（W 映射 vs 路由） | P1 | 中 | D1 |

> P0 = 上线必须；P1 = 应该做（体验/防回归/扩展）。

---

## 2. A 安全

### A1 [P0] 行级数据权限服务端强制
- **现状**：RBAC 的"操作"守卫（requireAction）服务端强制、可信；但"行"守卫没有。
  `CollReadController.java:15` 原话"数据范围过滤…此处返回全量（演示级）"；
  `GET /api/snapshot` 返回全部 34 集合**无 region 过滤**。user02（仅华北）登录后直接
  `GET /api/coll/dispatches` 或 `/api/snapshot` 能拿到**所有区域**的行 → 越权读数据。
- **方案**：
  1. 引入 `DataScopeFilter`：从 JWT Operator 读 `dataScopes`（平台管理员=全量；其余按 regions；
     司机/客户按各自绑定维度）。
  2. 在列表读取层（DataStore.list 或专门 ListService）对**含 region 的集合**
     （dispatches/contracts/plans/weighings/settlements/customers/terminals 等）按 region 过滤。
  3. `/api/snapshot` 与 `/api/coll/*` 都过 filter；`/api/logs` 按操作人可见范围过滤。
  4. 过滤实现方式取决于 B1：路 A（JSON）在应用层读出后 filter；路 B（规范化）走 SQL `WHERE region IN (...)`。
- **验收**：user02（华北）GET /api/coll/dispatches 只返回华北行；/api/snapshot 同口径；
  平台管理员全量；只读/客户/司机按各自范围；补 FlowIntegrationTest 断言（越权读被过滤）。

### A2 [P0] 登录防爆破 + 锁定（服务端）
- **现状**：`AuthService.login` 只有验证码 + bcrypt + 停用检查，**无失败计数/锁定**。
  M8 的"5 次锁 5 分钟"是**前端 localStorage**——换个浏览器就绕过。
- **方案**：Redis 计数（username + IP 双维度），连续 5 次失败锁 5 分钟；锁定态返回剩余秒数；
  成功清零。与前端 M8 口径对齐（前端锁定展示保留，服务端为权威）。
- **验收**：连续 6 次错误密码 → 第 6 次返回"已锁定 N 秒"；换浏览器/换 IP 段仍受 username 维度锁定；
  补集成测试（锁定/解锁/清零）。

### A3 [P1] 全局限流
- **现状**：全仓 0 处限流配置，任何端点可被无限刷（尤其登录、验证码）。
- **方案**：bucket4j（或 spring-cloud-gateway rate limit）；登录/验证码端点严格（如 10/min/IP），
  写端点适度（如 100/min/用户）。
- **验收**：超频返回 429 + Retry-After；补限流测试。

### A4 [P0] 密钥 / 配置外置
- **现状**：`application.yml` 硬编码 DB 密码（blms123456）+ JWT 演示密钥
  （`blms-demo-secret-key-change-me`，注释自述"生产必须外置"）。
- **方案**：`application-prod.yml` + 环境变量注入（DB 密码/JWT secret/Redis 地址）；
  JWT secret 外置 + 支持轮换（双密钥过渡期）；敏感项不进 git（.gitignore + 模板 `.env.example`）。
- **验收**：生产 profile 启动不依赖仓库内明文密钥；密钥轮换不中断在线会话。

### A5 [P1] 输入校验
- **现状**：全仓 0 处 `@Valid`/`@NotNull`；所有端点收 `Map<String,Object>` 手动取字段，
  无 schema 校验、无类型安全。
- **方案**：分阶段把高频写端点改为 `@Valid` DTO（createContract/createPlan/createDispatches/
  settlement 系列/admin 系列）；DTO 字段加 `@NotBlank`/`@NotNull`/`@Size`/`@Min`。
  暂不改的端点保留 Map 但补显式必填校验。
- **验收**：缺必填字段返回 400 + 字段级错误；补参数校验测试。

---

## 3. B 数据 / 架构

### B1 [P0] 存储模型决策（架构级，最大单项）
- **现状**：`biz_*` 表 = `id + payload JSON`（`V3__biz_tables.sql:1` 原话"整条记录 JSON payload，
  与前端 mock 同态"）。**这是文档库不是关系库**：无业务字段索引、无 SQL 关联/聚合、无引用完整性。
  上面 A1 行级过滤难做、B2 分页难做，根因都在这。
- **两条路**：
  - **路 A（快，保留 JSON）**：用 MySQL JSON 函数做行过滤 + 关键查询；对高频过滤字段
    （region/status/contractId 等）建**虚拟列 + 索引**。适合数据量 < 10w、查询模式固定。
    改动小，能先上线。
  - **路 B（彻底，规范化）**：核心表拆列（dispatches/contracts/plans/settlements/weighings/
    invoices 等），建索引 + 外键 + 唯一约束。工作量大，但支撑真规模 + 复杂查询 + 报表 + 行级过滤走 SQL。
- **建议**：**先路 A 上线**（配合 A1 行级过滤 + B2 分页），**路 B 作为后续演进**（按查询压力/报表需求触发，
  按表分批迁移，Flyway 版本化）。
- **验收**：路 A — 关键列表查询走索引、行级过滤正确、分页正确；路 B — 核心表 DDL + 迁移 + 查询改写 + 回归全绿。

### B2 [P1] 分页
- **现状**：控制器 0 处 page/size/offset；列表端点全量返回，前端客户端分页。
  演示 ~30 行无所谓，真产品几千条调度单就是每次拉全量。
- **方案**：列表端点加 `page/size`（默认 20），返回 `{list, total}`；`/api/snapshot` 保留
  （启动 hydrate 用）但**列表页改走分页端点**（不再拉全量）。
- **验收**：1000 条调度单，GET ?page=1&size=20 返回 20 条 + total=1000；前端列表页改分页加载。

### B3 [P0] 并发 / 冲突模型（架构级）
- **现状**：`DataStore` 单进程 `ReentrantLock` + `commitAll`；前端每次写后拉**整个 34 集合**快照
  = 单用户假设下的"最后写入胜出"，**无冲突检测**。两个操作员同时改同一张调度单，后拉快照者把前者的
  改动静默覆盖。不能水平扩展。
- **方案**：
  - **短期（单实例可接受）**：给可变记录加 `version` 字段，写端点**乐观锁**（version 不匹配 → 409），
    前端处理 409（提示"数据已变更，请刷新"）。
  - **长期（多实例）**：去全量快照，改**按资源读写**（GET 单条 + POST 变更，返回变更后的权威态）；
    分布式锁（Redis）或 DB 行锁；定时任务单实例（leader election，见 C4）。
- **验收**：两个会话改同一调度单 → 后者 409；多实例下无重复定时任务、无静默覆盖。

---

## 4. C 运维 / 可观测

### C1 [P0] 部署工件
- **现状**：无 `Dockerfile`/`docker-compose`；`application.yml` 硬编码（见 A4）。
- **方案**：多阶段 `Dockerfile`（maven build → jre runtime，非 root 用户，健康检查）+
  `docker-compose.yml`（app + mysql + redis，env 注入，健康检查依赖顺序）+ 生产 profile。
- **验收**：`docker compose up` 一键起全栈；/api/health 通过；重启后 Flyway 自动迁移。

### C2 [P0] 后端 CI
- **现状**：只有前端 `.github/workflows/ci.yml`（npm test + build）；后端无 CI。
- **方案**：后端 `.github/workflows/ci.yml`（setup-java 17 → mvn test → 可选 Docker build）；
  与前端 CI 并列。
- **验收**：push 触发 mvn test 全绿（141 断言）；构建产物可拉取。

### C3 [P1] 可观测性
- **现状**：无 actuator/health/metrics/tracing；日志只有 `com.blms: info`（非结构化）。
- **方案**：`spring-boot-starter-actuator`（/health /metrics /info，暴露关键指标）+
  结构化日志（logback JSON 格式，含 traceId）+ 可选 tracing（micrometer + OTLP）。
- **验收**：/actuator/health 返回 UP；metrics 可抓取；日志可被日志平台解析。

### C4 [P1] 服务端定时任务
- **现状**：`application.yml:39` `scheduler.auto-enabled: false`（靠前端轮询 /api/scheduler/tick）。
  真产品不能依赖每个浏览器实例驱动心跳。
- **方案**：生产开 auto-enabled（Quartz 或 xxl-job，**单实例 leader** 执行，避免多实例重复跑）；
  前端轮询降级为"兜底/演示"路径。
- **验收**：无浏览器时后端仍按时推进围栏/遥测/逾期/升级；多实例下任务只跑一次。

### C5 [P1] CORS / 部署拓扑
- **现状**：0 处 CORS 配置，靠 dev 代理（vue.config.js /api → 8081）。前后端分离部署时需明确拓扑。
- **方案**：生产 Nginx 反代（`/api` → 8081，`/` → dist 静态 + SPA fallback）；或 Spring CORS 白名单
  （限定前端域名）。推荐反代（无 CORS、同源、可加限流/日志）。
- **验收**：生产域名下登录/快照/写操作全通；跨域被正确限制。

---

## 5. D 开发体验

### D1 [P1] OpenAPI / Swagger
- **现状**：118 个端点无机器可读文档，前后端契约靠人工核对（2026-08-30 全量手工扫描）。
- **方案**：`springdoc-openapi-starter-webmvc-ui` 生成 `/v3/api-docs` + Swagger UI；
  关键端点补 @Operation/@Schema 注解（参数/返回/错误码）。
- **验收**：/v3/api-docs 可访问；118 端点有 schema；前端可据此生成类型。

### D2 [P1] API 契约测试（防漂移）
- **现状**：W 映射（前端 97 写端点）vs 控制器路由（后端 118）目前靠人工核对，漂移无 CI 拦截
  （上轮发现的 hashStr/派车顺序/契约漂移都是"翻译错"，根因是无自动契约检查）。
- **方案**：CI 静态检查——解析前端 W 映射（method+path+body 字段）vs 后端 @RequestMapping/@RequestBody
  字段，不匹配即红。把 2026-08-30 手工扫描固化成自动化。
- **验收**：故意加一个不匹配端点 → CI 红；当前全量匹配 → CI 绿。

---

## 6. 实施顺序（依赖图）

**Phase 1（安全底线 + 能部署，必须）**
`A4 密钥外置` → `A1 行级过滤` → `A2 防爆破` → `C1 部署工件` → `C2 后端 CI`
（先能安全地部署起来，再谈扩展）

**Phase 2（架构决定，决定能长多大）**
`B1 存储（先路 A）` → `B3 并发（乐观锁）` → `B2 分页`
（存储模型定了，行级过滤/分页/并发才有正确实现方式）

**Phase 3（体验 / 防回归 / 扩展）**
`A5 校验` → `C3 可观测` → `C4 定时任务` → `C5 CORS` → `D1 OpenAPI` → `D2 契约测试`

> 关键依赖：**B1 存储模型**是总开关——A1 行级过滤、B2 分页、B3 并发的实现方式都取决于它。
> 建议 Phase 2 先拍板 B1（路 A 先上线），其余随之落地。

---

## 7. 验收基线（"真产品"门槛）

- **安全**：行级数据权限服务端强制（A1）+ 登录防爆破（A2）+ 限流（A3）+ 密钥外置（A4）+ 关键端点校验（A5）
- **数据**：分页（B2）+ 乐观锁/冲突检测（B3）+ 核心查询走索引（B1 路 A）
- **运维**：docker 一键起全栈（C1）+ 后端 CI 全绿（C2）+ actuator 健康（C3）+ 服务端定时（C4）
- **开发**：OpenAPI 机器可读（D1）+ 契约测试防漂移（D2）

达到以上四条，即可从"演示/POC"过渡到"可上线的真产品"。

---

## 8. 风险与决策点

| 决策点 | 影响 | 建议 |
|---|---|---|
| **B1 存储：路 A vs 路 B** | 最大单项决定，影响 A1/B2/B3 全部实现方式 + 后续报表/查询 | 路 A 先上线（快），路 B 按压力分批演进 |
| **B3 并发：单实例+乐观锁 是否够用** | 取决于预期并发用户数 / 是否需水平扩展 | 先单实例+乐观锁；多实例需求明确再上分布式 |
| **前端内存引擎退役** | 真产品多用户后，local-first 内存引擎与"服务端权威"冲突（见前序讨论） | 触发条件=真实多用户生产；届时引擎收进 demo 门控，主路径薄客户端化 |
| **A5 校验改造范围** | 全量 DTO 化工作量大 | 先高频写端点，其余补显式必填校验 |

---

## 9. 进度记录

（实现时按 Phase 追加：任务 → 改动文件 → 验证输出 → done-verified）
