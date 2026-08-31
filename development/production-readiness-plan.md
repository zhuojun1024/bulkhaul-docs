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
| A1 | 安全 | 行级数据权限服务端强制 | **P0** ✅done | 中 | B1（过滤实现方式） |
| A2 | 安全 | 登录防爆破 + 锁定（服务端） | **P0** ✅done | 小 | — |
| A4 | 安全 | 密钥/配置外置（生产 profile） | **P0** ✅done | 小 | — |
| C1 | 运维 | 部署工件（Dockerfile + compose） | **P0** ✅done | 中 | A4 |
| C2 | 运维 | 后端 CI | **P0** ✅done | 小 | — |
| B1 | 架构 | 存储模型决策（JSON blob → 规范化） | **P0** | 大 | — |
| B3 | 架构 | 并发/冲突模型（乐观锁 / 多实例） | **P0** | 中 | B1 |
| A3 | 安全 | 全局限流 | P1 | 小 | A2 |
| A5 | 安全 | 输入校验（@Valid DTO） | P1 | 中 | — |
| B2 | 架构 | 分页 | P1 | 中 | B1 |
| C3 | 运维 | 可观测性（actuator + 结构化日志） | P1 | 小 | C1 |
| C4 | 运维 | 服务端定时任务（leader 单实例） | **P1** ✅done | 中 | B3 |
| C5 | 运维 | CORS / 部署拓扑（Nginx 反代） | **P1** ✅done | 小 | C1 |
| D1 | 开发 | OpenAPI / Swagger | **P1** ✅done | 小 | — |
| D2 | 开发 | API 契约测试（W 映射 vs 路由） | **P1** ✅done | 中 | D1 |

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
| **前端内存引擎退役（全量薄客户端化）** | 架构级：读路径从"快照 hydrate + 内存读"切到"按需 API 读"，flow.js（3720 行）整体退役；演示能力改由后端兜底 | **已拍板推进**（2026-09 讨论定稿）：演示数据后端化（seed_* + reset-demo 持久化）+ 双模式 flag 过渡 + 分 8 阶段实施，详见 §9 Phase 4 草案 |
| **A5 校验改造范围** | 全量 DTO 化工作量大 | 先高频写端点，其余补显式必填校验 |

---

## 9. 进度记录

（实现时按 Phase 追加：任务 → 改动文件 → 验证输出 → done-verified）

### Phase 1 安全（A1 / A2 / A3 / A4）— 2026-08-30

#### A1 行级数据权限服务端强制 — done-verified ✅
- **决策（用户拍板）**：扩大范围——过滤所有含区域维度的集合，前端同步验证。
- **实现**：
  - 新增 `service/admin/DataScopeService.java`（114 行）：
    - `scopeRegions()`：平台管理员→空（全量）；其余读 `dataScopes[username].regions`（空=全量）。
    - `filter(coll, rows)`：O(n) 预建 4 张区域映射（终端 id→region / 合同 id→loadTerminalId / 调度 id→loadTerminalId / 调度 id→contractId）。
    - 区域派生：dispatches/plans/contracts/transportRequests 直接 `loadTerminalId`；settlements 经 `contractId`；weighings 经 `dispatchId`（缺失再经其 contractId）。
    - `inScope(coll, rec)`：单条判定（无区域归属的记录可见，防御口径）。
  - `SnapshotController.snapshot()`：6 个区域集合过 `scope.filter`；其余集合与 logs 不过滤。
  - `CollReadController`：`list` 过滤 + `one` 越权范围返回 `403 forbidden`。
  - **不过滤（已论证）**：customers（region=省份 山西/陕西，非数据区域 华北/西北，过滤会全隐）、terminals（区域来源本身）、logs/messages（菜单/角色门控非区域维度）。
  - **前端零改动**：db 由后端快照 hydrate，快照已按操作人过滤 → 所有视图自动行级生效；`visibleDispatches()` 变为幂等（db 已预过滤）。
- **验证**：
  - 后端集成测试 +18 断言（user02 华北真子集 / 计划/合同/结算行级 / 越权 403 / 平台管理员全量 / 快照端点过滤 / 守卫）→ **PASS=159 FAIL=0**（原 141）。
  - 运行中后端 E2E：admin 202 调度单 vs user02 79（=华北装货侧 79，真子集）；plans 60→27 / contracts 40→16 / settlements 20→9 / weighings 365→149 / transportRequests 4→2；customers/logs 全量不过滤。
  - 前端 **npm test 556/0** + **verify-ui 82/0**（环节8 行级过滤 UI 断言全绿：分页总数=华北数、顶栏/列表"数据范围：华北"标签）。
- **提交**：bulkhaul-server `200743b`（已推送）。

#### A2 登录防爆破服务端强制 — done-verified ✅
- **决策（用户拍板）**：仅按用户名（Redis 按账号，5 次失败→锁 5 分钟，成功清零；对齐前端 M8 口径）。
- **实现**：
  - 新增 `auth/LoginLockoutService.java`（74 行）：Redis `login:fail:{user}`（计数，TTL 5 分钟滑动窗口）+ `login:lock:{user}`（锁定标记，TTL 5 分钟）；账号维度归一（trim+lowercase）；`clearAll()` 供演示自恢复。
  - `AuthService.login`：前置锁定拦截（code=locked，含剩余秒数）→ 凭据失败计数（第 5 次触发锁定）→ 成功清零。
  - `SnapshotController.resetDemo` 调 `lockout.clearAll()`：演示/测试自恢复（避免某账号被锁后影响后续场景登录）。
  - 前端 `views/login/index.vue`：处理服务端 `code=locked`（强制本地锁定展示，文案用服务端）；本地 M8 计数保留原口径（凭据/验证码失败均计入，体验层），服务端为权威。
- **验证**：
  - 后端集成测试 +A2 断言（5 次触发锁定 / 剩余递减 4→1 / 成功清零 / 大小写空白归一不绕过）→ **PASS=159 FAIL=0**。
  - 运行中后端 E2E：attempt1-4 credential（还剩 4/3/2/1 次）→ attempt5 locked（连续 5 次失败，已锁定 5 分钟）→ attempt6 locked（299 秒后可重试）；admin 不受影响。
  - **verify-ui 82/0**（场景15 M8 本地锁定 + 环节9 验证码登录 全绿）。
- **提交**：bulkhaul-server `200743b` + bulkhaul-manage-web `f5aace5`（均已推送）。

#### A4 密钥/配置外置 — done-verified ✅
- **实现**：
  - `application.yml` 全部敏感项改环境变量占位符（`DB_URL/DB_USERNAME/DB_PASSWORD/REDIS_HOST/REDIS_PORT/JWT_SECRET/JWT_TTL_MINUTES/SERVER_PORT/SCHEDULER_AUTO_ENABLED/LOG_LEVEL_COM_BLMS`），保留 dev 默认值 → **dev/测试零行为变化**（mvn test 159/0 验证）。
  - 新增 `.env.example`（生产环境变量模板，含 JWT_SECRET 生成提示 `openssl rand -base64 48`）。
  - 新增 `application-prod.yml`（生产 profile：敏感项无 dev 默认，缺 env 即启动失败，避免误用演示密钥上线；scheduler 默认开）。
  - `.gitignore` 忽略 `.env`。
- **验证**：mvn test 全绿（占位符默认值生效）；生产部署路径 = 注入 env 变量 + `SPRING_PROFILES_ACTIVE=prod`。
- **提交**：bulkhaul-server `200743b`（已推送）。

#### A3 全局限流 — done-verified ✅
- **实现**（Redis 固定窗口，非引入新依赖；单实例 B3 口径下 Redis 计数即权威）：
  - 新增 `common/RateLimitService.java`：Redis 固定窗口（60s）计数，**Lua 原子 INCR+EXPIRE**（无竞态）；脚本返回单字符串 `计数:剩余秒数`（规避 StringRedisTemplate 反序列化 Lua 多 bulk 表失败）；**Redis 不可用 fail-open**（限流是防护层，不阻断全部请求）；`clearAll()` 供 reset-demo 自恢复。
  - 新增 `common/RateLimitFilter.java`（`OncePerRequestFilter`，**在 JwtAuthFilter 之后**运行——写档可按用户限流，未认证按 IP 兜底）：
    - **登录档**（`/api/auth/login`、`/api/auth/captcha`）：按 IP（严格，防刷登录/验证码）。
    - **写档**（`/api/**` 的 POST/PUT/DELETE）：按用户（适度，防单账号高频写）。
    - **GET 不限**（读多写少，快照/列表高频轮询不受限）。
    - **排除**：`/api/scheduler/tick`（前端每 3s 心跳，系统行为非用户写）+ `/api/admin/reset-demo`（自恢复端点）。
  - `SecurityConfig`：`addFilterAfter(RateLimitFilter, JwtAuthFilter.class)`。
  - 配置（A4 口径：dev 宽松 / prod 严格）：`blms.rate-limit.{enabled,login-per-minute,write-per-minute}`；dev 默认 120/min（登录）+ 600/min（写）——**verify-ui 24 次登录 + 大量写不触发**（联调/演示零行为变化）；prod（application-prod.yml）10/min（登录，按 IP）+ 100/min（写，按用户）；`.env.example` 补 `RATE_LIMIT_*`。
  - `SnapshotController.reset-demo`：补 `rateLimit.clearAll()`（与 A2 `lockout.clearAll()` 并列自恢复，避免某 IP/账号被限后影响后续场景）。
  - 超限 → **429 + `Retry-After`（秒）+ `{ok:false, error:"请求过于频繁，请稍后再试", code:rate_limited}`**（前端 fetch 层不崩，仅该写失败）。
- **验证**：
  - 后端集成测试 +A3 断言（未超限放行 / 超限返回 Retry-After 1..60s / 不同维度独立 / 写档独立 / clearAll 自恢复）→ **PASS=168 FAIL=0**（原 163）。
  - 运行中后端 E2E（真 HTTP，X-Forwarded-For 注入测试 IP `10.77.77.77` 不污染真实计数）：连发 121 次验证码 → 第 121 次 **429 Retry-After=59** `code=rate_limited`；真实 IP（127.0.0.1）不受假 IP 影响（admin 仍正常登录）；reset-demo 自恢复（清空后假 IP 重新放行 200）。
  - 前端 **verify-ui 82/0**（A3 过滤器在真实 HTTP 路径上，24 次登录 + 大量写零误报，过滤器顺序/维度正确）。
- **提交**：bulkhaul-server（本条，已推送）。

#### A5 输入校验 — done-verified ✅（阶段一：3 个高频 create 端点 DTO 化）
- **实现**（分阶段：先 DTO 化高频 create 端点，其余 94 个写端点保留 Map + 服务层显式校验，后续按需推进）：
  - 新增 `dto/CreateContractRequest` / `CreatePlanRequest` / `CreateDispatchesRequest`（`@NotBlank`/`@NotNull`/`@Positive`/`@Min`，约束消息对齐服务层既有校验文案，UX 不变；可选字段缺省走服务层默认值，`toMap()` 非 null 才放）。
  - `ContractController.createContract/createPlan` + `DispatchController.create` 改 `@Valid @RequestBody <DTO>`（字段与前端表单 1:1，已核对 contract/create.vue、plan/create.vue、dispatch 表单）。
  - `GlobalExceptionHandler`：`MethodArgumentNotValidException` → **400 + `code:validation_error` + `data.fieldErrors`（字段→首条消息，字段级）**；`ApiResult.fail` 补 `(error, code, data)` 重载。
  - 契约：**已认证用户 + 非法 body → 400**（匿名 → 401 属认证层，非 A5 范畴）；校验先于业务逻辑（非法 body 不进 service）。
- **验证**：
  - 后端集成测试 +`ValidationIntegrationTest`（MockMvc，admin 登录取 token）：contract 缺 name+quantity / quantity=0 / plan 缺 contractId / dispatch 缺 planId / count=0 → 均 **400 validation_error + 字段级 fieldErrors**；必填齐全（占位 ID）→ 通过 @Valid 进业务逻辑（非 validation_error）→ **Tests run 23（17+6）全绿**。
  - 运行中后端 E2E（真 HTTP，admin token）：13 条断言全过（400 + code + 字段级 name/quantity/contractId/planId/count + 必填齐全非 400）。
  - **构建提速**：WSL 原生副本（`~/bulkhaul-server-wsl`）编译/测试（规避 /mnt/d 跨文件系统慢 IO），`dev` 分支提交、验证后 squash 合并 master（单 commit）。
- **提交**：bulkhaul-server（本条，dev → master 单 commit，已推送）+ bulkhaul-manage-web（verify-a5-validation.mjs）。

### Phase 2 架构（B1 / B3）— 2026-08-30

#### B1 存储模型规范化（路 B）— done-verified ✅
- **决策（用户拍板）**：路 B 彻底规范化（用户覆盖推荐的路 A——核心表全量拆列，非仅加索引）。
- **实现**：
  - 新增 `db/migration/V5__core_tables_normalize.sql`：7 核心表（biz_dispatches/contracts/plans/transportRequests/settlements/weighings/invoices）各加 `version INT NOT NULL DEFAULT 1` + `region VARCHAR(32) NULL` + `status` + 关键外键列（contract_id/load_terminal_id/dispatch_id/settlement_id）+ 索引（region/status/外键）；从 payload 回填 status+外键，region 经多级 JOIN（装货侧终端所在数据区域）回填。
  - `DataStore.syncDerivedColumns(coll)`：commitAll 时回填 version/region/status/外键派生列（region 为派生列，经终端/合同/调度 JOIN 回填；无匹配保持 NULL = 无区域 → 数据范围可见，与 DataScopeService 防御语义一致）。
- **验证**：mvn test **PASS=159 FAIL=0**（基线不变，派生列同步不影响业务逻辑）；region 回填为 A1 行级过滤（`WHERE region IN`）与 B2 分页提供 SQL 基础。
- **提交**：bulkhaul-server `e5d390e`（已推送）。

#### B3 并发/冲突模型（单实例 + 乐观锁）— done-verified ✅
- **决策（用户拍板）**：单实例 + 乐观锁（version 字段，不匹配 → 409，前端处理 409"数据已变更，请刷新"）。
- **实现**：
  - 新增 `common/OptimisticLockContext.java`（请求级 ThreadLocal 期望版本，等价 Operator.current() 读 SecurityContext 的模式）+ `common/OptimisticLockException.java`（→409）+ `common/OptimisticLockSupport.java`（expectFromBody/expectFromQuery，controller 便捷入口）。
  - `DataStore.commitAll()`：写锁内 drain 期望版本逐条比对——不匹配 → 先从 DB 恢复该记录为权威态（撤销本次孤儿改动，保持内存与 DB 一致）再抛 OptimisticLockException（→409）；匹配 → version+1 随 payload 持久化。未登记（直接 service 调用 / 定时任务 / 未参与乐观锁的写）→ 空 map → no-op（保持既有"最后写入胜出"）。
  - `DataStore.ensureCoreVersions`：核心集合记录持久化前确保 payload 携带 version（运行时新建记录 V6 迁移覆盖不到，此处兜底）。
  - `GlobalExceptionHandler`：补 OptimisticLockException → 409 conflict（此前缺失）。
  - controller 接线：6 核心集合写端点登记 expectedVersion（dispatches：confirmLoad/accept/depart/arrive/confirmUnload/cancel/reassign/resume；contracts：change/extend/terminate/complete/archive；plans：cancel；settlements：startReconcile/recalc/customerConfirm/customerObjection/confirmSettle/recordPayment/revertPayment/applyPrepayment/dunning/issueInvoice/redFlush；weighings：correct）。无 body 端点用 `?expectedVersion` 查询参数，有 body 端点用 body.expectedVersion；缺省 → 不参与（兼容旧客户端 / 直接 service 调用）。
  - 新增 `db/migration/V6__core_payload_version_seed.sql`：既有核心记录 payload 注入 version=1（缺失才注入，已有不覆盖）——前端经快照读 payload 的 version 发 expectedVersion；payload 无 version → 前端不发 → B3 不触发（本迁移补上端到端闭环）。
  - 前端 `api/index.js`：afterWrite 对 6 核心集合写注入 expectedVersion（记录型读 a[0].version；weighing 按 id 查 db.weighings）；409（code=conflict）派发 `blms:conflict` window 事件（api 层不依赖 element-plus，保持 node 可测）。
  - 前端 `main.js`：监听 `blms:conflict` → ElMessage.warning 展示后端文案（"数据已变更（…），请刷新后重试"）；1.5s 去抖避免 toast 叠加。
- **验证**：
  - 后端集成测试 +B3 断言（版本匹配 → 提交成功且 version 递增 / 不匹配 → OptimisticLockException（→409）/ **无静默覆盖**（DB 保持会话 A 权威态，非会话 B 覆盖）/ 恢复后基于新版本可继续提交）→ **PASS=163 FAIL=0**（原 159）。
  - 运行中后端 E2E（真 HTTP，两会话改同一调度单 PD-00067）：会话 A（expectedVersion=1）200 version 1→2；会话 B（过期 expectedVersion=1）**409 conflict**"数据已变更（PD-00067 版本 1 → 2），请刷新后重试"；无静默覆盖（DB 保持会话 A 的 V001/D001 version 2，非会话 B 的 V005/D005）。
  - 前端 **npm test 556/0** + eslint 0 + build 成功（api 层保持 node 可测）。
- **提交**：bulkhaul-server + bulkhaul-manage-web（本条，已推送）。

#### B2 分页 — 后端 done-verified ✅ / 前端列表页改分页 ⏸（架构决定，待拍板）
- **后端（已实现+验证）**：
  - `CollReadController.list`：`GET /api/coll/{name}` 加 `page/size`（size 默认 20、上限 200）→ `{list, total, page, size}`；**不带 page → 全量返回（向后兼容）**——`/api/snapshot` hydrate、验证脚本、旧前端客户端分页均不受影响。
  - 行级数据范围（A1）与分页叠加：区域集合先按当前操作人过滤，`total`=过滤后行数（user02 华北 total=79 < admin 202，页内全为华北）。
- **验证**：
  - 后端集成测试 +B2 断言（不带 page 全量向后兼容 / page=1 size=20 → 20 条+total / page=2 无重叠 / 超范围页空 list+total 不变 / size 上限 200 / 行级+分页 user02 total=华北）→ **PASS=174 FAIL=0**（原 168）。
  - 运行中后端 E2E（真 HTTP）：不带 page 全量 202；page=1 size=20 → 20+total=202；page=2 无重叠；超范围页空；size 上限 200；user02 total=79 < admin 202（行级+分页正确）。
- **前端（待拍板）**："列表页改走分页端点（不再拉全量）"= **前端内存引擎退役的首步**（§8 决策点：触发条件=真实多用户生产，届时引擎收进 demo 门控、主路径薄客户端化）。当前前端为 local-first 内存引擎（`/api/snapshot` hydrate 全量 34 集合 → 客户端分页 `filtered.slice`，详情/交叉引用/流程引擎依赖全量内存态）。改列表页走分页端点需薄客户端化列表视图（分页加载 + 详情按需取单条），属架构级改动，**不随 B2 后端默认推进**。
- **提交**：bulkhaul-server（本条，已推送）。

#### C1 部署工件（Dockerfile + compose 全栈）— done-verified ✅
- **实现**：
  - `bulkhaul-server/Dockerfile`：Maven 多阶段（pom 预拉依赖层缓存 + -DskipTests）→ JRE 非 root 运行（8081，JAVA_OPTS 基线）+ `.dockerignore`。
  - `bulkhaul-manage-web/Dockerfile`：node:20 构建 → nginx:1.27 托管 + `.dockerignore`。
  - `bulkhaul-manage-web/deploy/nginx.conf.template`：官方镜像 envsubst 机制（BACKEND_UPSTREAM 占位符）；/api 反代后端（X-Forwarded-For/Proto）；SPA 兜底；gzip + hash 资源 30d 长缓存；HEALTHCHECK。
  - `docker-compose.yml`（**workspace root**，两仓库公共父目录）：mysql:8 + redis:7 + backend + frontend，depends_on healthcheck 顺序编排（mysql/redis healthy → backend healthy → frontend）；敏感项经 env 注入（A4）；命名卷持久化。
  - `.env.example`（root + docs 副本）+ `docs/deployment/`（README 部署指南 + compose/env 参考副本）。
- **验证（2026-09，WSL2 原生 Docker 端到端起栈）**：WSL2（Ubuntu 24.04 + systemd）装 **Docker Engine 29.7.2**（static bundle，systemd 托管，daemon 配 HTTP(S)_PROXY 拉镜像）+ **compose v2 v5.5.0**（CLI plugin）+ **iptables/nftables**（WSL 缺，bridge NAT 必需）。`docker compose up -d` 全栈起：**mysql:8 + redis:7 + backend + frontend 四容器全 healthy**；后端 `/api/health` `db:connected tables:105`、actuator UP、**Flyway V1-V6 已应用**、OpenAPI 200；**前端 nginx 托管 Vue + `/api` 反代后端 200**（验证码真实返回）。**修掉一个真实 bug**：`nginx.conf.template` 原 `upstream` 块在 nginx **启动时**用系统 DNS 解析 `backend`（Docker/WSL2 下易超时失败 → `host not found in upstream` → nginx emerg 退出）→ 改 **`resolver 127.0.0.11` + 变量 upstream 运行时解析**（web `9fa02b9`）。注：本环境 8080 被宿主推理服务占用，本地 compose 副本改 8082 验证（仓库保持 8080 默认）。
- **提交**：server `88abe4a` + web `033335e` + **web `9fa02b9`（nginx 运行时 DNS 修复）** + docs（本条）已推送。

> Phase 1 完成度：A1 ✅ / A2 ✅ / A3 ✅ / A4 ✅ / **A5 ✅（阶段一：3 个高频 create 端点 DTO 化，其余写端点分阶段推进）**。

#### C2 后端 CI（GitHub Actions）— done-verified ✅
- **实现**：
  - `bulkhaul-server/.github/workflows/backend-ci.yml`：push/PR 到 master 触发；
    services `mysql:8`（建 blms_test 库 + healthcheck）+ `redis:7`（healthcheck）；JDK 17（temurin）+ Maven 依赖缓存；`mvn -B test`（Flyway V1-V4 建表灌种子 + 159 断言）；失败上传 surefire 报告。
  - `application-test.yml` 数据源改 `TEST_DB_*` 环境变量占位符（本地默认值不变，零行为变化）；CI 经 env 指向 service 容器。
- **验证（2026-09，GitHub Actions 实际运行）**：server `backend-ci` **8/8 全绿**（含最新 `d3dab8a`，push 自动触发，mvn test 全过）。**web `CI` 曾红两次**（`f593367`/`9fa02b9`）——根因：D2 契约步骤检出后端仓库到**子目录** `bulkhaul-server/`，原 `../bulkhaul-server` 指向仓库外（不存在）→ 后端路由 0 → 97 端点全判缺失 → 红。修复：CI 改 `./bulkhaul-server` + 脚本目录缺失显式报错 exit 2（web `39f5984`），**CI 转绿**。前后端 CI 均实际运行验证。
- **提交**：server `696521e` + web `39f5984`（CI 路径修复）已推送。

#### C3 可观测性（actuator + 结构化日志）— done-verified ✅
- **实现**（actuator + 结构化 JSON 日志含 traceId；tracing/OTLP 可选，本环境无 collector 默认关）：
  - `pom`：+ `spring-boot-starter-actuator`（health/info/metrics）+ `logstash-logback-encoder 8.0`（JSON 日志）。
  - `application.yml`：`management.endpoints.web.exposure.include: health,info,metrics` + `metrics.tags.application: bulkhaul-server`。
  - 新增 `common/TraceIdFilter.java`（`@Order(HIGHEST_PRECEDENCE)`，**先于安全过滤链**）：请求级 traceId 写 MDC（取上游 `X-Request-Id` 或生成 16 位短 UUID），回写响应头供前端/监控关联；401/403 等安全日志也带 traceId。
  - 新增 `logback-spring.xml`：**prod → 结构化 JSON**（logstash，含 `@timestamp`/`message`/`logger_name`/`level`/MDC `traceId` + 静态 `app:bulkhaul-server`，可被 ELK/Loki 解析）；**!prod → 人类可读 pattern**（含 traceId，本地调试）。
  - `SecurityConfig`：`/actuator/health` + `/actuator/info` 放行（公开探活）；`/actuator/metrics` 保持已认证（受保护，可抓取）。
  - 既有 `/api/health`（自定义，含表数）保留不动（compose 后端 healthcheck 为 TCP 探活，不受影响）。
- **验证**：
  - 后端集成测试 +`ActuatorIntegrationTest`（MockMvc）：`/actuator/health` 200 UP / `/actuator/info` 200 / `/actuator/metrics` 未认证 401（受保护）→ **Tests run 26（17+6+3）全绿**。
  - 运行中后端 E2E（真 HTTP）：9 条全过（health 公开 200 UP / info 200 / metrics 未认证 401 + 已认证 200 含 JVM 指标 / 既有 /api/health 向后兼容 / X-Request-Id 回写 + 缺失生成 16 位）。
  - **prod profile 日志格式实测**：`-Dspring.profiles.active=prod` 启动，日志行均为合法 JSON（`{"@timestamp":...,"message":...,"logger_name":...,"level":...,"app":"bulkhaul-server"}`），可被日志平台解析。
  - **构建提速**：WSL 原生副本（`~/bulkhaul-server-wsl`）编译/测试（规避 /mnt/d 慢 IO），在线经 aliyun 镜像拉新依赖（actuator/micrometer/logstash-encoder）；`dev` 分支提交、验证后 squash 合并 master（单 commit）。
- **提交**：bulkhaul-server（本条，dev → master 单 commit，已推送）+ bulkhaul-manage-web（verify-c3-observability.mjs）。

> Phase 2：B1 ✅（路 B 规范化，V5 拆列 + 派生列同步）/ B3 ✅（单实例 + 乐观锁，version 不匹配 → 409 + 无静默覆盖）/ **B2 分页：后端 ✅（page/size → {list,total}，向后兼容）/ 前端列表页改分页 ⏸（=内存引擎退役首步，待拍板）** / C1 🔶（工件就绪，起栈验证待 Docker 环境）/ C2 🔶（CI workflow 就绪，运行验证待 push 触发）/ **C3 ✅（actuator + 结构化 JSON 日志含 traceId）**。

### Phase 3 体验 / 防回归 / 扩展（C4 / C5 / D1 / D2）— 2026-09

#### C4 服务端定时任务（单实例 leader 租约）— done-verified ✅
- **决策（自主，按上线标准）**：复用既有 `@Scheduled(fixedDelay=3000)` + `autoEnabled` 机制（生产 `application-prod.yml` 已开 `auto-enabled: true`，`@EnableScheduling` 已就绪）——"无浏览器时后端仍按时推进"已满足；**补齐缺失的"单实例 leader 执行"**（多实例下任务只跑一次）。选 **Redis leader 租约**（A3 已验证的单字符串 Lua 原子 SET/RENEW 模式，轻量、无新依赖，优于 Quartz/xxl-job 重量级方案）。
- **实现**：
  - 新增 `service/scheduler/SchedulerLeaderService.java`：Redis leader 租约（Lua 原子 SET/RENEW，返回单字符串 `1`/`0`——A3 验证过的单 bulk 反序列化模式）；**TTL 10s**（>3s tick 间隔，leader 每轮续期；leader 宕机后 ≤10s 由其余实例接管，自愈）；**Redis 不可用 fail-close**（跳过本轮，宁停勿双跑，避免重复围栏/遥测/升级）；`clear()` 供 reset-demo 自恢复。
  - `SchedulerService.runSchedulerTick`：`autoEnabled` 守卫后加 `leader.tryAcquireOrRenew()`（非 leader → 跳过 `{skipped,reason:not_leader}`）；**手动 `/api/scheduler/tick` 走 `doTick()` 不受 leader 限制**（演示/验证确定性）。
  - `SnapshotController.reset-demo`：补 `leader.clear()` 自恢复（清空租约，避免旧租约残留影响接管）。
  - 前端 3s 轮询（`mock/scheduler.js`）降级为**兜底/演示路径**（浏览器态调 `/api/scheduler/tick` + 快照刷新），生产以服务端 leader 定时为准。
- **验证**：
  - 后端集成测试 +`SchedulerLeaderServiceTest`（获取/续期/他实例持租约非 leader/clear 后接管）→ **PASS=174 FAIL=0**（Tests run 27=17+6+3+1）。
  - 运行中后端 E2E（`verify-c4-scheduler.mjs`，真 HTTP）：手动 tick 返回统计非 skipped / reset-demo note 提及 leader 自恢复 / reset-demo 后 tick 仍正常 → **5/5 全绿**。
- **提交**：bulkhaul-server（本条，dev → master 单 commit）+ bulkhaul-manage-web（verify-c4-scheduler.mjs）。

#### C5 CORS / 部署拓扑（Spring CORS 白名单显式启用 opt-in）— done-verified ✅
- **决策（自主，按上线标准）**：主部署拓扑走 **Nginx 反代**（C1：`/api` → 8081 同源）。**关键修正**：反代会**转发浏览器 Origin 同时改写 Host**，后端看到 Origin≠Host 即判跨域——若"默认拒绝跨域"会**误伤同源反代**（verify-ui 8086 代理 / 生产 Nginx 均中招，登录 POST 被 403 拦截 → verify-ui 登录超时）。故 CORS 采用**白名单显式启用（opt-in）**：默认空=**不做 CORS 处理**（同源反代直通，跨域由浏览器同源策略限制——无 ACAO 头浏览器拦截跨域读，认证端点 token 在 Authorization 头非 Cookie 跨域无法携带→401）；配置白名单（`CORS_ALLOWED_ORIGINS`，逗号分隔+通配）=**启用 CORS 强制**（仅白名单来源可跨域，纵深防御，用于前端与后端不同源部署的显式场景）。
- **实现**：
  - 新增 `auth/CorsConfig.java`：`CorsConfigurationSource` Bean——**allowed-origins 空 → 返回空 source（不做 CORS 处理，请求直通）**；非空 → `setAllowedOriginPatterns`（通配）+ `allowCredentials=true`，注册 `/api/**` + `/actuator/health` + `/actuator/info`。
  - `SecurityConfig`：`http.cors(Customizer.withDefaults())`（**预检 OPTIONS 先于认证**，避免被 401 拦截）。
  - `application.yml` / `application-prod.yml`：`blms.cors.allowed-origins=${CORS_ALLOWED_ORIGINS:}`（默认空=不做 CORS 处理）；`.env.example` 补 `CORS_ALLOWED_ORIGINS`。
- **验证**：
  - 后端全量回归（CORS opt-in 改动后）→ **PASS=174 FAIL=0**（BUILD SUCCESS）。
  - 运行中后端 E2E（`verify-c5-cors.mjs`，真 HTTP）：同源（无 Origin）200 无 ACAO / 跨域（未授权 Origin）无 ACAO / 预检 OPTIONS 无 ACAO / 跨域不影响同源功能（captcha 200）→ **5/5 全绿**。
  - **回归修复验证**：原"默认拒绝跨域"导致 verify-ui（同源反代，Origin 8086≠Host 8081）登录 POST 被 403 拦截 → `login()` 15s 超时（确定性复现 3 次）；改 opt-in 后 **verify-ui 82/0 全绿**（同源反代直通，跨域仍被浏览器同源策略限制）。
- **提交**：bulkhaul-server（本条，dev → master 单 commit，含 opt-in 修正）+ bulkhaul-manage-web（verify-c5-cors.mjs）。

#### D1 OpenAPI / Swagger（springdoc 机器可读契约 + 关键端点注解）— done-verified ✅
- **决策（自主，按上线标准）**：`springdoc-openapi-starter-webmvc-ui 2.6.0`（Boot 3.3.5 兼容，aliyun 镜像可用）；**dev 公开**（联调看文档）/ **生产需认证**（`blms.openapi.public`，默认 prod=false，不暴露 API 面）。
- **实现**：
  - `pom.xml`：+ `springdoc-openapi-starter-webmvc-ui 2.6.0`。
  - 新增 `common/OpenApiConfig.java`：全局 OpenAPI（标题/版本/描述 + **全局 JWT Bearer 安全方案**）。
  - `SecurityConfig`：`blms.openapi.public` 开关——dev 公开 `/v3/api-docs/**` + `/swagger-ui/**`，生产 false 需认证；`application.yml`/`application-prod.yml` + `.env.example` 补 `OPENAPI_PUBLIC`。
  - 关键端点 `@Operation`（auth captcha/login/me、coll 列表/单条、scheduler tick、contract/plan 新建）+ A5 DTO `@Schema`（必填/可选/示例）。
- **验证**：
  - 后端全量回归（springdoc 编译 + 注解）→ **PASS=174 FAIL=0**（BUILD SUCCESS，MVN_EXIT=0）。
  - 运行中后端 E2E（`verify-d1-openapi.mjs`，真 HTTP）：`/v3/api-docs` 200 OpenAPI 3.x（47KB）/ 标题正确 / **paths ≥100（118 端点覆盖）** / `@Operation` summary（login/contract）/ `CreateContractRequest` schema 含 name+quantity 且 name required / Swagger UI 可访问 / **`/api` 仍需认证（无 token 401，OpenAPI 公开未放行业务）** → **11/11 全绿**。
- **提交**：bulkhaul-server（本条，dev → master 单 commit）+ bulkhaul-manage-web（verify-d1-openapi.mjs）。

#### D2 API 契约测试（防漂移，前端 W 映射 vs 后端路由，CI 拦截）— done-verified ✅
- **决策（自主，按上线标准）**：CI 静态检查——解析前端 `src/api/index.js` W 映射（method+归一化 path）vs 后端 `*Controller.java` 路由（`@RequestMapping`+`@Get/Post/Put/DeleteMapping`），**前端每个写端点必须在后端存在**（`{...}` 路径变量归一为 `{x}`），缺失/方法不符即红（exit 1）。把 2026-08-30 手工扫描（hashStr/派车顺序/契约漂移都是"翻译错"）固化成自动化。
- **实现**：
  - 新增 `scripts/check-contract.mjs`（web 仓库）：前端 W 映射 97 写端点（method+path）vs 后端 116 路由；前端缺失/方法不符 → 红；后端未被 W 引用的读/认证/快照路由为孤儿（仅提示不红）。
  - `package.json`：`test:contract` 脚本；`ci.yml`：检出后端仓库（`zhuojun1024/bulkhaul-server`）+ 契约测试步骤（漂移即红）。
- **验证**：
  - **当前全量匹配 → 绿**：前端 **97/97** W 写端点全部在后端匹配（method+path），0 漂移（exit 0）。
  - **红路径验证**（验收"故意加一个不匹配端点 → CI 红"）：临时注入 `fakeDriftEndpoint: /api/nonexistent/drift-check` → **FAIL exit 1**（检出 1 个契约漂移）；还原 → 97/97 绿。
  - 前端 **npm test 556/0**（verify-flow 不受影响）。
- **提交**：bulkhaul-manage-web（本条，dev → master 单 commit）。

> **Phase 3 完成度**：C4 ✅（服务端定时任务，Redis leader 租约单实例执行）/ C5 ✅（CORS 白名单默认拒绝跨域，Nginx 反代同源主拓扑）/ D1 ✅（springdoc OpenAPI 3.x + Swagger UI，dev 公开/生产认证，118 端点 schema）/ D2 ✅（API 契约测试，前端 97 W 端点 vs 后端 116 路由，CI 拦截漂移，红/绿路径均验证）。
> **全部 15 项缺口**：A1✅ A2✅ A3✅ A4✅ A5✅(阶段一) / B1✅ B2✅(后端) B3✅ / **C1✅（WSL2 原生 Docker 端到端起栈验证通过，nginx 运行时 DNS 修复）** **C2✅（前后端 CI 实际运行全绿，web CI 契约路径修复）** C3✅ C4✅ C5✅ / D1✅ D2✅。**Phase 4 已拍板（2026-09）：阶段 0 重置持久化已完成（reset-demo 限管理员 + commitAll 回写 biz_* 持久化 + 前端 admin 门控 + AuditLog 冲突修复），阶段 1-7 待推进。**

### Phase 4 全量薄客户端化（B2 前端 + 内存引擎退役）— 已拍板（2026-09 讨论整理；三点决策已定，自阶段 0 起实施）

#### 背景与现状（代码核实，非估计）
- **写路径已是薄客户端**：97 个写端点全部委托后端（`FlowCtx` 638 行 + `DispatchService` 510 / `ContractService` 565 / `SettlementService` 562 / `AdminService` 433 等；乐观锁 B3 / RBAC B1 / 审计 A1 均在服务端）。flow.js **不再承担业务逻辑**。
- **读路径仍是 local-first**：`/api/snapshot` 一次性 hydrate 全量 34 集合进内存 → 之后所有读走内存。flow.js 现存 3720 行 / 263 处 db 读 = 读缓存 + 乐观 UI（点操作立即改本地，API fire-and-forget 200ms 防抖）+ 客户端聚合。
- **前端消费面**：37 视图，**33 个直接读 `db.*`（180 处）**，29 个 import flow；本地聚合：`dashboard.js`（31 处 db 读）/ `report.js`（18 处）/ flow.js 展示派生值（结算量/车次成本等）。
- **后端聚合逻辑已 1:1 搬完但未暴露**：`DashboardService`（93 行，kpi/safeDays，注释"与前端 dashboard.js 1:1"）+ `ReportService`（280 行，monthly/customer/commodity/terminal/cost 五报表，口径与前端一致）——**无任何 controller 引用**（缺的只是接线）。
- **演示数据已持久化**：`seed_*` 只读快照表（V4 固化种子态，不受 `biz_*` 回写污染）+ `POST /api/admin/reset-demo`（内存重置 + 清防爆破/限流/leader 租约）+ 前端头像菜单按钮（现有）。
- **数据模型**：每集合 = `(id, payload JSON)` + 少量派生列（B1 路 B）；跨集合关系在 payload 内 id 引用；**聚合是 Java 内存 join，非 SQL 多表 JOIN**（演示量级 202 调度单/40 合同/60 计划完全够快）。

#### 决策 1：全量薄客户端化（内存引擎整体退役）
- **终态**：前端 = 常规 CRUD 项目（页面 → 数据层 → /api），flow.js 退役；DB 为唯一权威，无客户端/服务端漂移，多用户读一致，行级范围每次请求服务端过滤。
- **演示能力由后端兜底**（决策 2）：演示 = 连服务端 + 一键重置恢复演示版本，**放弃离线演示**（可接受：演示场景本就有服务端）。
- **双模式过渡**：feature flag——演示模式 = 现有内存引擎（快照 hydrate + 本地引擎，现有 556 npm 断言 + 82 verify-ui E2E 继续有效）；生产模式 = 薄客户端。按页灰度切换，迁完翻转默认，最后移除引擎。

#### 决策 2：演示数据重置持久化（Phase 4 前置，独立可交付）
- **现状缺口**：`reset-demo` 只重置内存（接口 note 原话"仅内存/Redis，不回写 DB"）——旧架构下正确（内存权威），**薄客户端化后 DB 权威 → 重置不回写则重启后脏数据复活**。
- **改动**：`reset-demo` 增加 `commitAll()`（种子态回写 `biz_*`，派生列同步）→ 重置 = **持久化恢复演示数据版本**；更新 note；补测试（重置后 `biz_* == seed_*`、重启后仍种子态）+ E2E（reset → snapshot 对比种子）。
- **权限**：限**管理员角色**（`RequireAction`，演示用 admin 账号；非管理员 403）。
- **前置验证（风险点）**：`seed_*` 是 V4 时点快照——若 V5/V6 后种子数据演进过（如 v5 客户运输需求 transportRequests），快照可能缺表/缺数据 → `tryLoadSeedFromSnapshot` 整体失败回退"启动时内存捕获"（脏库则演示版本被污染）。**动手前先核对 `seed_*` 覆盖全部 34 集合且与当前种子一致**；不一致 → 出新迁移重新固化快照。
- **附带收益**：演示数据集从此**版本受控**（以后加演示场景 = 更新种子 + 迁移固化，重置即恢复新版本）。

#### 决策 3：数据层 + 聚合端点（"常规 CRUD"的正确形态）
- **不是"页面直接 import /api 各查各的"**（强交叉引用域会得到 N 并行请求 + 视图间不一致 + 无 loading/error 态 + 重复请求爆炸）。常规做法 = 先有数据层：
  - **前端数据层**：`useCollection(name)` composable（fetch + 缓存 + 写后失效 + 乐观更新）——页面 import 数据层，不直接 fetch。
  - **后端接线（薄）**：暴露 `DashboardService`/`ReportService`（逻辑已 1:1，只差 controller）。
  - **详情聚合端点（真·补）**：详情页 join 7-8 集合（dispatch+vehicle+driver+plan+contract+weighing+settlement+exception）→ 新增 `GET /api/dispatch/{id}/detail`（Java 内存组装，非 SQL join）。
  - **`/api/snapshot` 保留**：演示模式 hydrate / E2E 脚本 / 重置后整页重拉均用。
- **SQL 下推不在本次范围**：数据量上到几万~十万行时内存聚合才需下推 SQL（抽派生列/视图/索引）——那是数据模型演进，不计入本次迁移成本。

#### 分阶段实施（每阶段独立可交付 + 绿门槛：mvn test / npm test / verify-ui / E2E / 契约测试）
0. **重置持久化**（独立，现架构下也正确）：核对 `seed_*` 完整性 → `reset-demo` 加 `commitAll()` → 测试 + E2E → 提交推送
1. **后端接线**：dashboard/report controller + 详情聚合端点 + 测试（口径与前端 1:1 对拍）
2. **前端数据层**：`useCollection` composable + 缓存/失效（尚不改视图）
3. **列表页 → 分页**（B2，后端已就绪）——首个切生产模式的页面
4. **详情视图**（跨 join）→ 详情聚合端点
5. **看板/工作台** → dashboard/report 端点异步化
6. **flow.js 门控**：feature flag 收进演示模式，主路径移除本地态/守卫（后端已权威）
7. **测试套件重建**：556 npm 断言 + 82 verify-ui E2E 围绕 API mock / 真服务端；reset-demo 作为场景间前置恢复继续可用

#### 成本与风险（修正后结构）
- **主战场在前端**（数据层 + 33 视图 / 180 处 db 读迁移）；**后端轻**（接线 + 1-2 个聚合端点，逻辑已搬完）。
- 37 视图 × 异步化 = 高回归风险 → 分阶段 + 每阶段绿门槛 + 双模式 flag 灰度（出问题时"重置 + 重拉"是干净逃生通道，演示模式随时可回退）。
- 代价：离线演示能力丢失（演示需连服务端）；点击→往返延迟（WSL 几十 ms，生产可感知但可接受）；迁移期双套代码并存（flag 门控）。

#### 决策（3 点，已拍板 2026-09）
1. **重置权限**：✅ 限**平台管理员**（后端 `requireAction('admin')` + 前端 `can('admin')` 按钮门控；非管理员 403 兜底）。
2. **双模式 flag 默认值**：✅ **过渡期演示模式默认**（现有内存引擎 + 556 npm / 82 verify-ui 继续有效），**切完列表页（阶段 3）后翻转**为生产模式默认，最后移除引擎。
3. **起步顺序**：✅ **从阶段 0（重置持久化，独立低风险）开始**。

#### 进度
- **阶段 0 重置持久化（2026-08 完成）**：
  - `V7__seed_core_payload_version.sql`：把 V6 的 `payload.version` 镜像到 `seed_*` 7 核心表（否则重置后 B3 乐观锁首写不触发）。
  - `SnapshotController.resetDemo()`：`requireAction('admin')` 门控 → `resetToSeed()` → **`commitAll()` 回写 `biz_*`（持久化）** → 清防爆破/限流/leader 租约 → 审计日志；note 更新为"已回写 DB（持久化）"。
  - 前端 `Navbar.vue`：重置按钮 `v-if="can('admin')"`（仅平台管理员可见）+ 调 `POST /admin/reset-demo` → 清本地快照残留 → `refreshDb()` 重拉种子态 → 整页刷新。
  - 测试：`FlowIntegrationTest` 新增 `@Order(92) phase4_resetDemo_persist`（脏写→非管理员 403 不变→admin 重置→`biz_*` 持久化回种子→内存恢复→审计留痕）。
  - **附带修复（既有缺陷）**：`AuditLog` 多 Spring 上下文共享 `op_log` 时 seq 落后 → 主键冲突（`DuplicateKeyException`，本地全量测试偶发红，CI Linux 顺序不同故绿）。改为冲突时从库内最大值 `resyncSeq()` 重同步 + 有界重试，与上下文创建顺序无关。
  - 验证：后端 `mvn test` 28/28 绿（含 P4 + AuditLog 修复）/ 前端 `npm test` 556/0 / 契约 97/97 / `npm run build` 通过 / 运行栈 reset-demo admin 200（note 含 leader）+ user02 403 / verify-ui 主链路（确认装货）通过。
- **阶段 1 后端接线（2026-08-31 完成，done-verified）**：
  - DashboardService 扩展：commodityStructure / modeShare / terminalThroughput / vehicleStatus / workbenchStats（todayDispatches/todayLoad/todayUnload/monthSettled/yesterday*/prevMonthSettled）/ workbenchTodoList（contract/dispatch/exception/settlement/overdue 五类待办）。
  - 新增 ReportController：GET /api/dashboard/kpi、/api/dashboard/charts（四图）、/api/workbench/stats、/api/workbench/todos、/api/report/monthly|customer|commodity|terminal|cost（五报表，ReportService 逻辑 1:1 暴露）。
  - 新增 DispatchDetailService + GET /api/dispatch/{id}/detail：join dispatch+commodity+vehicle+driver+loadTerminal+unloadTerminal+weighings+contract+plan+settlements+exceptions，派生 settleQty/qualityDeduction（FlowCtx 口径）；不存在 404。
  - 测试：Phase4AggregationTest（5 测试 / 39 断言，端点值 vs DataStore 独立重算对拍：kpi/charts/workbench/report/dispatchDetail 五组 parity）。
  - 验证：mvn -o test 33/33 绿（Flow 18 PASS=184 + Phase4 5 PASS=39 + Validation 6 + Actuator 3 + Scheduler 1）；契约 97/97（新增 9 个读端点按读端点孤儿口径计，不红）。
- **阶段 2 前端数据层（2026-08-31 完成，done-verified）**：
  - src/composables/collectionStore.js：模块级 reactive 缓存（复合 key：name + 查询签名，同集合不同查询不串缓存）；getCollection/setRows/invalidate/invalidateMany/invalidateAllFor/optimisticUpdate/setLoading/setError/resetStore。
  - src/composables/useCollection.js：useCollection(name, opts|()=>opts) → {data,loading,error,total,refresh,update,invalidate}；带 page → 服务端分页 {list,total}；不带 → 全量（可带 status/mode/keyword/dateFrom/dateTo 过滤）；node/演示（USE_API=false）镜像本地 db + 本地过滤（口径与后端 CollReadController.applyFilters 一致）。
  - src/api/index.js：WRITE_COLL 域映射 + invalidateForWrite(fnName)（写后按域失效 + 复合 key 前缀失效）；refreshDb 成功后派发 blms:refreshed 事件（生产模式页面重取权威态）。
  - 测试：scripts/verify-collection.mjs（20 断言：store 纯逻辑 10 + useCollection node 模式 10，含分页切片/复合 key 独立/过滤口径）；npm test 556/0 无回归。
- **阶段 3 合同列表生产模式（2026-08-31 完成，done-verified）**：
  - feature flag：src/mode.js——appMode()/isProduction()/setMode/clearModeOverride；localStorage blms_app_mode > 构建默认 DEFAULT_MODE（过渡期 demo；本阶段验证通过后翻转为 production，按决策 2）。
  - 后端 /api/coll/{name} 扩展可选过滤（CollReadController.applyFilters）：status/mode 字段等值、keyword（id/name 子串，contracts 特例含发货方/收货方客户名，与前端 filtered 口径一致）、dateFrom/dateTo（signDate 区间）；无参向后兼容全量；FlowIntegrationTest B2 用例调用点同步 8 参签名。
  - views/contract/list.vue 生产模式：PROD=isProduction() 时行/总数/过滤/分页全走 useCollection('contracts', ()=>({page,size,status,mode,keyword,dateFrom,dateTo,key:'contracts:list'})) 服务端权威；watch([page,pageSize,filter]) 重取 + blms:refreshed 重取；演示模式（默认）保持本地 filtered/paged 客户端分页（现有断言不变）；交叉引用列（find.*）经启动 hydrate 的本地 db 渲染（生产模式仍 hydrate，db 供交叉引用）。
  - E2E 场景 20（verify-ui）：生产模式（localStorage 覆盖）admin 登录 → 合同列表分页总数=后端全量、交叉引用列渲染客户名、状态 chip 过滤总数=后端过滤数（服务端分页+过滤+交叉引用三断言）。
  - 验证：mvn -o test 33/33 / npm test 556/0 / npm run test:collection 20/0 / npm run build 通过 / 运行栈端点实测（full 40、paged total=40 list=5、status=executing 15=15、keyword=HT 40=40、date2026 34=34、paged+status total=15、客户名 keyword 4=4）/ verify-ui 含场景 20 全绿。DEFAULT_MODE 翻转为 production（决策 2：切完列表页后翻转）。
  - **阶段 3 绿门槛终验（2026-08-31，全绿）**：DEFAULT_MODE 翻转为 production 后复跑全部门槛——verify-ui **85/0**（场景 1-20 全绿，含场景 20 生产模式三断言；场景 6 合同运输需求页签/979 详情不受翻转影响，因仅合同列表页 opt-in isProduction）/ npm test 556/0 / test:collection 20/0 / 契约 97/97 / build 通过。
  - **E2E 两处既有敏感性修复（非本阶段回归，详见 lessons-learned §9）**：
    - 环节6（DND 标记）：消息列表按最新在前排序，定时任务生成的异常消息把唯一系统消息挤出第 1 页 → 断言前先按"系统"类型筛选（.filter-bar .el-select），标记断言与排序解耦；另标记依赖本地 db.dnd，需等 3s 定时任务快照刷新把已落库 dnd 同步回本地（200ms 防抖刷新可能早于 PUT 落库而回写种子态）。
    - 场景13（P2 在途）：验证后端须 SCHEDULER_AUTO_ENABLED=false（确定性，node 侧手动驱动 tick）；重建脚本误用生产默认 true → 后台 3s tick 把新在途车次转 exception。recreate_backend.sh/start_backend.sh 已固化 false。
 - **阶段 4 调度详情生产模式（2026-08-31 完成，done-verified）**：
   - views/dispatch/detail.vue 生产模式：PROD=isProduction() 时详情读面（dispatch/commodity/vehicle/driver/loadTerminal/unloadTerminal）onMounted/换单时 GET /api/dispatch/{id}/detail 单次往返取后端权威态（阶段 1 已建聚合端点，Java 内存组装非 SQL join）；写入仍走 flow（乐观改 detail.dispatch + afterWrite 落库，W 映射仅取 a[0].id 故端点对象兼容）；磅单读 db.weighings（flow 乐观 push + refreshDb 同步，与端点 weighings 同源）；时间线/打印 HTML 的场站引用同步改用端点派生 computed（loadTerminal/unloadTerminal）。不监听 blms:refreshed 重取，避免 200ms 防抖刷新早于 PUT 落库回写种子态覆盖乐观态（与阶段 3 同口径，导航时重取权威态）。
   - E2E 场景 21（verify-ui）：生产模式（localStorage 覆盖）admin 登录 → 调度详情头部渲染调度单号 + 商品名（读面来自聚合端点）+ 磅单行数=后端该单磅单数（车牌断言按车辆口径条件触发，非公路车次跳过）共 4 断言。
   - **阶段 4 绿门槛终验（2026-08-31，全绿）**：mvn -o test 33/33 / npm test 556/0 / npm run test:collection 20/0 / 契约 97/97 / npm run build 通过 / verify-ui **89/0**（场景 1-21 全绿，含场景 21 生产模式调度详情 4 断言；DEFAULT_MODE 已为 production，演示路径由 localStorage 未设 production 时回退覆盖，现有场景不受影响）。
 - **阶段 5 看板/工作台生产模式（2026-08-31 完成，done-verified）**：
   - views/dashboard/monitor.vue 生产模式：PROD=isProduction() 时 KPI（utilization/onTimeRate/safeDays/customerCount）+ 四图（commodityStructure/modeShare/terminalThroughput/vehicleStatus）onMounted 走 /api/dashboard/kpi + /api/dashboard/charts（后端权威，不依赖本地 db 聚合）；KPI 的 totalVolume/totalRevenue 由种子随机历史趋势 volumeTrend 派生（后端无对应口径）→ 取本地 dashboard.kpi 合并；环比趋势（volumeTrendPct/revenueTrendPct）随之取本地。
   - views/dashboard/workbench.vue 生产模式：指标卡（todayDispatches/todayLoad/todayUnload/monthSettled + 环比）onMounted 走 /api/workbench/stats、待办列表走 /api/workbench/todos（后端权威）；天气/问候为按日期确定性派生的演示数据源（非业务态）→ 两模式取本地；欢迎横幅 3 个速览数（executingContracts/intransitCount/monthVolume/pendingExceptions）为 hydrated-db 交叉引用速览（与阶段 3 交叉引用同口径）→ 两模式取本地。
   - **设计决策（趋势 derive vs static，按经验定）**：历史趋势（volumeTrend 12 月/exceptionTrend 30 天）为种子随机历史数据，**非 db 可派生**（种子仅含当期业务态，无历史运量/异常序列）→ 两模式均取本地 dashboard（诚实标注：属演示历史，非业务态）；不伪造后端口径。db 派生的实时指标（KPI/四图/工作台指标/待办）生产模式走后端聚合端点（阶段 1 已建，口径 1:1 对拍）。
   - E2E 场景 22（verify-ui）：生产模式（localStorage 覆盖）fetch 监听证明薄客户端确实调用 /workbench/stats + /workbench/todos + /dashboard/kpi + /dashboard/charts（db 派生指标两模式值相同，故用调用断言证明异步化生效）+ 工作台"今日调度"卡值=后端 todayDispatches + 看板 KPI 区渲染（6 断言）。
   - **阶段 5 绿门槛终验（2026-08-31，全绿）**：mvn -o test 33/33 / npm test 556/0 / npm run test:collection 20/0 / 契约 97/97 / npm run build 通过 / verify-ui **95/0**（场景 1-22 全绿，含场景 22 生产模式看板/工作台 6 断言；DEFAULT_MODE 已为 production，演示路径由 localStorage 未设 production 时回退覆盖，现有场景不受影响）。
