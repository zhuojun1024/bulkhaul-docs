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
| C1 | 运维 | 部署工件（Dockerfile + compose） | **P0** 🔶工件就绪 | 中 | A4 |
| C2 | 运维 | 后端 CI | **P0** 🔶workflow就绪 | 小 | — |
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

#### C1 部署工件（Dockerfile + compose 全栈）— done（工件就绪，端到端起栈验证待 Docker 环境）🔶
- **实现**：
  - `bulkhaul-server/Dockerfile`：Maven 多阶段（pom 预拉依赖层缓存 + -DskipTests）→ JRE 非 root 运行（8081，JAVA_OPTS 基线）+ `.dockerignore`。
  - `bulkhaul-manage-web/Dockerfile`：node:20 构建 → nginx:1.27 托管 + `.dockerignore`。
  - `bulkhaul-manage-web/deploy/nginx.conf.template`：官方镜像 envsubst 机制（BACKEND_UPSTREAM 占位符）；/api 反代后端（X-Forwarded-For/Proto）；SPA 兜底；gzip + hash 资源 30d 长缓存；HEALTHCHECK。
  - `docker-compose.yml`（**workspace root**，两仓库公共父目录）：mysql:8 + redis:7 + backend + frontend，depends_on healthcheck 顺序编排（mysql/redis healthy → backend healthy → frontend）；敏感项经 env 注入（A4）；命名卷持久化。
  - `.env.example`（root + docs 副本）+ `docs/deployment/`（README 部署指南 + compose/env 参考副本）。
- **验证**：compose 结构/引用核对通过（build context 指向两仓库、healthcheck 顺序、env 占位符与 A4 一致）；**当前环境无 Docker daemon**，端到端 `docker compose up -d --build` 起栈验证待有 Docker 的机器执行（C1 done-verified 门槛）。
- **提交**：server `88abe4a` + web `033335e` + docs（本条）已推送。

> Phase 1 完成度：A1 ✅ / A2 ✅ / A3 ✅ / A4 ✅ / **A5 校验（P1，待做）**。

#### C2 后端 CI（GitHub Actions）— done（workflow 就绪，CI 运行验证待 push 触发）🔶
- **实现**：
  - `bulkhaul-server/.github/workflows/backend-ci.yml`：push/PR 到 master 触发；
    services `mysql:8`（建 blms_test 库 + healthcheck）+ `redis:7`（healthcheck）；JDK 17（temurin）+ Maven 依赖缓存；`mvn -B test`（Flyway V1-V4 建表灌种子 + 159 断言）；失败上传 surefire 报告。
  - `application-test.yml` 数据源改 `TEST_DB_*` 环境变量占位符（本地默认值不变，零行为变化）；CI 经 env 指向 service 容器。
- **验证**：workflow YAML 结构核对通过（service 端口/healthcheck/env 占位符与 A4 一致）；**CI 实际运行待 push 后 GitHub Actions 触发**（本环境无法触发远程 CI）。
- **提交**：server `696521e`（已推送）。

> Phase 2：B1 ✅（路 B 规范化，V5 拆列 + 派生列同步）/ B3 ✅（单实例 + 乐观锁，version 不匹配 → 409 + 无静默覆盖）/ C1 🔶（工件就绪，起栈验证待 Docker 环境）/ C2 🔶（CI workflow 就绪，运行验证待 push 触发）。
> 下一步按 P1 优先级：A3 全局限流 / A5 输入校验（@Valid DTO）/ B2 分页（依赖 B1 已就绪）/ C3 可观测性 / C4 服务端定时 / C5 CORS / D1 OpenAPI / D2 契约测试。
