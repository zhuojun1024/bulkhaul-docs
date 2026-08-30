# 后端架构规划（bulkhaul-manage-web → 完整后端服务）

> 2026-08-26 基于前端代码检索制定。前端现状：37 个视图、44 条路由、mock 服务层 150 个函数、
> verify-flow 554 断言 + verify-ui 82 断言。本文是后端实施的唯一规划来源。

## 一、前端关键部分盘点（规划依据）

### 1. 数据模型 = 37 个集合（src/mock/base.js `db`）
商品/客户/场站/车辆/司机/合同/运输需求/计划/调度/磅单/仓库/库存/结算/收款/预付/应付/催收/
银行流水/发票/消息/异常/事故/培训/检查/用户/角色/运价卡/保险理赔/日志 + 配置类
（fenceConfig、dataScopes、dnd、safetyStocks、escalateConfig）。
→ 未来 37 张表，结构经 10 个版本迭代已稳定，可直接反推 DDL。

### 2. 服务层 = 150 个导出函数（src/mock/flow.js，3622 行）
每个函数 = 一个未来 API endpoint，且已带完整服务端语义：
- `requireAction('xxx')` RBAC 单点校验（默认拒绝，未登录态拦截）
- `logAction()` 审计留痕（含失败日志）
- `version` 字段乐观锁（车辆/司机，validateResourceCommit 提交前二次校验）
- `createDispatches` 事务化（资源不足整体失败，无半套残留）
- 金额 `round()` 定点舍入（后端改 BigDecimal）

### 3. 权限体系（src/permission-table.js）
7 角色 × 24 操作码，菜单级 + 按钮级双层，运行时可编辑（db.rolePerms 数据化，角色管理页可改）。

### 4. 鉴权（flow.js login）
验证码（一次性 60s）+ SHA-256 密码哈希 + 账号状态校验；注释明确"对接后端时换 bcrypt + JWT"。

### 5. 五个"心跳"函数（前端轮询触发 → 后端定时任务）
checkFenceEvents（电子围栏）/ advanceTelemetry（在途轨迹）/ recalcOverdueAll（逾期重算）/
escalatePendingExceptions（异常超时升级）/ escalateContractApprovals（审批超时催办）。

### 6. 测试资产
verify-flow.mjs 554 断言（环节 1–35）+ verify-ui.mjs 82 断言 = 现成的后端验收规格书。

## 二、技术选型

JDK 17 + Spring Boot 3 + Maven + MySQL 8 + Redis 7 + MyBatis-Plus + Spring Security + JWT

| 选择 | 理由 |
|------|------|
| Spring Boot 3 + JDK17 | Ubuntu 24.04 原生支持，生态最成熟，用户点名 Java |
| MyBatis-Plus（非 JPA） | 结算/对账大量精确金额计算与聚合查询，需直接控制 SQL 与 BigDecimal 精度；MP 的 @Version 乐观锁、逻辑删除与前端已有 version 字段一一对应；JPA 懒加载/脏检查在"函数式服务层"迁移模式下反而碍事 |
| MySQL 8 | 37 张表强外键关联（合同→计划→调度→磅单→结算），事务型数据库刚需；CTE/窗口函数满足三报表 |
| Redis | 验证码存储（替代前端 captchaStore Map）、JWT 失效/刷新、登录限流、消息未读计数 |
| 不引入 MQ | 单实例演示级，Spring @Scheduled 足够 |
| 不拆微服务 | 业务域虽多但数据强耦合（一次调度联动 6 个集合），单模块多包 |

## 三、包结构（按业务域分包，与前端 mock 模块 1:1 对应）

```
com.blms
├── auth/          登录/验证码/JWT/RBAC 注解与切面（requireAction → @RequireAction AOP）
├── common/        统一响应/异常/审计切面（logAction → AuditAspect）/ID 生成
├── domain/        37 张表的 entity + mapper
├── service/
│   ├── contract/  合同+审批链+变更+运价卡
│   ├── plan/      计划+拆车
│   ├── dispatch/  调度执行+磅单+资源占用（事务核心）
│   ├── warehouse/ 库存+FIFO+安全库存
│   ├── settlement/结算+对账+收款+预付+银行核销+催收
│   ├── invoice/   开票+红冲
│   ├── exception/ 异常+事故+保险理赔
│   ├── safety/    培训+检查+证照过期
│   ├── portal/    客户门户（需求/确认/异议）
│   ├── driverapp/ 司机端（接单/扫码/签收）
│   ├── system/    用户/角色/日志
│   └── report/    三报表+看板
└── job/           5 个定时任务（围栏/轨迹/逾期/升级/催办）
```

## 四、关键设计决策

1. **API 与 flow.js 函数 1:1 映射**——前端迁移成本最低：mock 服务层换 HTTP client，
   函数签名即接口契约，37 个视图几乎不用改
2. **RBAC 单点校验用 AOP**：`@RequireAction("dispatch")` 切面，复刻 requireAction
   "默认拒绝"语义；权限数据存 role/permission 表（保留运行时可编辑能力）
3. **审计用 AOP 切面**：写操作自动落 op_log 表，含失败记录（前端 logAction 的 result 字段）
4. **金额全部 BigDecimal**，禁用 double——前端 round() 是演示妥协，后端必须精确
5. **事务边界 = 前端"函数"边界**：createDispatches 类多集合写操作一个 @Transactional；
   乐观锁冲突返回 409 让前端重试（复刻 validateResourceCommit）
6. **ID 生成**：genId 的"前缀+最大序列+1"模式 → 每表独立 sequence 表，保证删除不复用
7. **测试移植**：verify-flow.mjs 554 断言按环节 1–35 逐条移植为 JUnit 5 +
   Testcontainers(MySQL) 集成测试，直接对比前后端行为一致性

## 五、实施阶段（每阶段独立可验证）

| 阶段 | 内容 | 验证标准 | 状态 |
|------|------|----------|------|
| 1 | 环境 + 骨架：JDK/Maven/MySQL/Redis 安装，Spring Boot 工程，Flyway 建 37 张表（DDL 从前端种子数据反推） | 服务启动 + 表结构检查 | **done-verified**（2026-08-26，见下） |
| 2 | 鉴权 + RBAC：登录/JWT/验证码 + @RequireAction 切面 + 审计切面 | 登录 + 越权拦截测试 | **done-verified**（2026-08-26，verify-auth 22/22） |
| 3 | 主链路：合同→计划→调度→磅单→仓储（事务核心） | 环节 1–12 断言移植 | **done-verified**（2026-08-26，主链路 44/44 + 结算 35/35 + 辅助 34/34，见下） |
| 4 | 财务：结算→对账→收款→发票→银行核销 | 环节 13–25 断言移植 | **done-verified**（2026-08-27，verify-settlement 35/35 + verify-phase45 财务段，见下） |
| 5 | 辅助域：异常/安全/保险/门户/司机端/报表 + 管理后台 CRUD/客户门户/运价卡 | 环节 26–35 断言移植 + 管理域端点 | **done-verified**（2026-08-29，安全/保险/审批 verify-phase45 55/55 + 异常/磅单 verify-aux 34/34 + 管理 CRUD/客户门户/运价卡 verify-admin 51/51，见下） |
| 6 | 定时任务 + 全量测试 + 前后端联调 | 全部断言全绿 + 前端切真实 API | **partial**（2026-08-29，5 心跳 verify-scheduler 10/10；7 脚本 252 断言全绿；**前端切真实 API 未做**） |

### 阶段 1 实施记录（2026-08-26，done-verified）
- 工程位置：`D:\Documents\workbench\bulkhaul\bulkhaul-server`（com.blms:bulkhaul-server:0.1.0）
- 环境：JDK 17.0.20 / Maven 3.8.7 / MySQL 8.0.46 / Redis 7.0.15（apt 安装，systemctl 管理）
- 数据库：blms（utf8mb4_unicode_ci），用户 blms/blms123456（localhost + 127.0.0.1）
- **服务端口 8081**（8080 被用户本机 llama-server 占用，勿改回）
- DDL：前端 `scripts/dump-schema.mjs` dump 种子字段 → `scripts/gen-ddl.mjs` 反推 →
  `src/main/resources/db/migration/V1__init.sql`（36 张表：31 业务 + fence_config/
  escalate_config 配置 + sys_role_perm/user_dnd/user_data_scope 权限KV；金额 DECIMAL(16,2)、
  嵌套结构 JSON 列、统一 created_at/updated_at 审计列）
- 验证：`mvn compile` 通过；Flyway V1 success；应用启动 4.6s；
  GET /api/health → `{"tables":37,"db":"connected","status":"UP"}`
- 骨架：BulkhaulServerApplication（@EnableScheduling + @MapperScan）/ SecurityConfig
  （最小版：STATELESS + /api/health 与 /api/auth/** 放行）/ HealthController
- 遗留：`bulkhaul-backend/` 为早期尝试（仅 2 文件，无 DDL），已被 bulkhaul-server 取代，**已删除**（2026-08-26 用户确认）

### 阶段 2 实施记录（2026-08-26，done-verified）
- 鉴权链路（与前端 flow.js 1:1）：
  - `GET /api/auth/captcha` → { id, code, svg }（Redis 存储，60s TTL，一次性消费；演示环境回传 code 供自动化）
  - `POST /api/auth/login` → { ok, token, user } | { error, code: captcha|credential|disabled }（bcrypt 密码，username 或 phone 登录）
  - `GET /api/auth/me` → 当前操作人（JWT Bearer）
- 组件：JwtService(HS256, 480min) / JwtAuthFilter(STATELESS) / CaptchaService(Redis) /
  RbacService(sys_role_perm 数据化优先 → ROLE_ACTIONS 内置兜底 → 默认拒绝；5min 缓存 + allowAll 集合) /
  @RequireAction + RequireActionAspect(拦截→审计 fail + 403；放行→执行→审计 success) /
  AuditLog(op_log 表，LOG- 前缀序列不复用，上限 1000 裁剪) / GlobalExceptionHandler(403/401/500 JSON)
- 种子：V2__seed_auth.sql（98 用户 bcrypt(123456) / 8 角色 / 8 角色权限 / 数据范围 / 围栏+升级配置）
  由 `scripts/gen-seed-auth.mjs` 从前端 mock dump 生成（前端 scripts/ 下）
- 验证：`bulkhaul-server/scripts/verify-auth.mjs` **22/22 PASS**（真实 HTTP：登录 7 场景、JWT 3 场景、
  RBAC 放行/拦截/未登录 4 场景、审计落库 5 场景）
- 踩坑记录：ConcurrentHashMap 不允许 null 值（actions=NULL 全放行角色需单独 allowAll 集合）；
  WSL 会话退出会杀 mvn 进程（用 terminal background 模式保活）；8080 被 Windows 侧 llama-server 占用

### 阶段 3 实施记录（2026-08-26，done-verified）

**移植架构（关键决策）**：前端 flow.js 是纯内存操作（改共享 reactive `db`，数据模型=扁平对象数组+嵌套结构）。
后端采用「内存数据仓库 + JSON 文档表」，保证 554 断言精确通过且 1:1 移植机械化：
- `DataStore`：启动时从 `biz_*` 表全量加载 34 集合到内存（等价前端共享 `db`），业务逻辑对 `Map` 操作
  （与前端对数组操作同构）；写操作后 `commitAll()` 按脏集合回写（DELETE+批量 INSERT）；粗粒度写锁保「单写者」语义
- `FlowCtx`：共享上下文 + 确定性算法（genId/tareOf/loadVarianceOf/calcSettlementFees/rollup/warehouseOut-In/
  occupyResource 逐行翻译）；运行时随机值用 ThreadLocalRandom（断言均为区间/关系断言，非精确值）
- 分域 Service（函数名与 flow.js 1:1）：contract / dispatch / settlement / weighing / exception / warehouse

**数据种子**：V3__biz_tables.sql（34 集合 biz_* 表，id 标量 + payload JSON 整条记录），
由 `scripts/gen-biz-seed.mjs` 从前端 mock 全量 dump 生成（后端与前端演示同态，448KB）。

**已实现域（与前端 1:1）**：
- 合同/计划：createContract（信用校验+守卫）/ createPlan / 合同回卷
- 调度（状态机全生命周期）：createDispatches（事务化两阶段+拆车均摊+资源占用）/ confirmLoad（M3 库存守卫+进磅）/
  depart / arrive / confirmUnload（出磅+质检+仓储入库+应付生成+回卷）/ cancel / reassign / 司机端 accept/depart/arrive/
  signReceipt/supplementReceipt/scanConfirmLoad/scanConfirmUnload
- 结算：settlementCandidates / generateSettlements / startReconcile（三方比对）/ recalc / customerConfirm /
  customerObjection / confirmSettle（签收硬拦截）/ recordPayment（超收截断）/ revertPayment / dunning /
  collectPrepayment / applyPrepayment / issueInvoice / redFlushInvoiceRow / markInvoiceStale
- 磅单：correctWeighing（复磅+结算联动：重算金额、已对账回待对账、已开票标记陈旧）
- 异常：reportException（事故类生成事故记录）/ resumeDispatch（exceptionFrom 精确回原态）/ acceptException /
  finishException / closeException（结算补扣+事故结案+车辆转维护）
- 仓储：manualInbound（守卫+批次+库存）/ safetyStockOf / availableStockOf / inventoryAlerts

**验证（真实 HTTP，非 mock）**：
- `scripts/verify-mainflow.mjs` **44/44 PASS**：合同→计划→调度→装货→发车→到达→卸货，含拆车/磅单/质检/仓储/回卷/审计/持久化
- `scripts/verify-settlement.mjs` **35/35 PASS**：candidates→generate→对账→客户确认→结算→收款→发票，含签收硬拦截+补签联动
- `scripts/verify-aux.mjs` **34/34 PASS**：磅单更正联动结算 + 异常处置全生命周期 + 手工入库守卫

**踩坑记录**：
- Jackson 把 JSON 数字反序列化成 `Long`，`(int) c.get("progress")` 裸强转 `Long→int` 抛 ClassCastException
  → progress 统一存 `int`（`Math.round` 返回 long 须 `(int)` 转型后 put），判断用 `intNum`（对 Integer/Long/Double 都安全）
- 结算联动断言须取**出磅**磅单（结算量按出磅净重算，改进磅不影响金额）
- 验证脚本选合同须查**剩余量>0**（quantity 大但已被计划占满的合同会触发 quantity>0 守卫）
- GlobalExceptionHandler 把 500 吞成 `data=null`，排障须另写探针抓真实 error 字段

### 阶段 4/5/6 实施记录（2026-08-26 实现，2026-08-27 复验全绿）

**已实现域（阶段 4 财务 + 阶段 5 辅助 + 阶段 6 定时任务）**：
- 合同审批链/变更审批（阶段 5）：submitContractApproval / approveContract（两级链，非末级/末级）/
  rejectContract / changeContract（改价转审批 pendingChange）/ approveContractChange / rejectContractChange
- 安全域（阶段 5）：registerAccident / closeAccident / addTraining / completeTraining /
  addInspection / 证照过期计算（driverLicenseExpired / vehicleInspectionExpired）
- 保险域（阶段 5）：fileInsuranceClaim / assessInsuranceClaim / settleInsuranceClaim（冲减事故损失）/
  rejectInsuranceClaim
- 财务核销（阶段 4）：generatePayables / payPayable / addBankStatement / autoMatchBank /
  matchBankRecord / 超余额拦截
- 定时任务（阶段 6）：SchedulerService 5 心跳（advanceTelemetry / checkFenceEvents /
  recalcOverdueAll / escalatePendingExceptions / escalateContractApprovals），`@Scheduled(fixedDelay=3000)`
  自动轮询 + `POST /api/scheduler/tick` 手动端点（doTick 无开关守卫，确定性触发）
- 读取面：`/api/coll/*` 全集合读取端点（CollReadController）+ 报表/看板（ReportService / DashboardService）

**验证（真实 HTTP，2026-08-27 全量复跑 201/201 PASS）**：
- verify-auth 23/23（含 2026-08-27 修复：probe 端点 @RequireAction 补回 + 用户不存在登录审计场景）
- verify-mainflow 44/44 / verify-settlement 35/35 / verify-aux 34/34
- verify-phase45 55/55（合同审批 13 + 合同变更 11 + 安全 8 + 保险 12 + 财务核销 11）
- verify-scheduler 10/10（tick 统计 + 遥测推进 + 围栏延误异常 + 异常升级 + 逾期校准 + 审批催办 + 幂等）

**2026-08-27 修复记录（复验时发现并修复的 3 个回归/脚本缺陷）**：
1. **probe 端点丢失 RBAC 守卫**：DispatchController.probe 缺 `@RequireAction`（阶段 2 有，后续改动丢失），
   user16 只读用户未被 403 拦截 → 补 `@RequireAction(value="dispatch", module="调度管理", action="下发调度单")`
2. **自动 tick 与验证竞态**：`@Scheduled(fixedDelay=3000)` 默认开启，种子 15 个在途车次 eta 均为
   2026-08-26（次日全部超期），自动 tick 把在途车次转 exception，导致 verify-scheduler intransit=0、
   verify-mainflow 新发车次被围栏抢先转异常 → `blms.scheduler.auto-enabled` 默认改 **false**
   （前端联调时前端自己轮询 /api/scheduler/tick；需后台心跳时 `--blms.scheduler.auto-enabled=true`）
3. **验证脚本硬编码日期过期**：verify-phase45（培训/事故/银行流水日期 2026-08-26）与 verify-scheduler
   （超 ETA 阈值 2026-08-26 23:00）→ 全部改动态日期（TODAY/NOW 运行时生成）；
   verify-auth 补"用户不存在"登录失败场景（登录失败审计断言自给自足，不再依赖历史累计）

**验证运行规程（重要，避免状态漂移误判）**：
- 6 个脚本**必须按 auth → mainflow → settlement → aux → phase45 → scheduler 顺序**在**同一份种子态**上跑
- 脚本会消耗种子资源（mainflow 占用合同剩余量/车辆司机，scheduler 消费在途车次），跑完一轮后
  再跑 mainflow/scheduler 会因资源耗尽失败——**重跑前先重置库**：
  `sudo mysql -e "DROP DATABASE IF EXISTS blms; CREATE DATABASE blms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"`
  然后重启服务（Flyway 重放 V1–V3 重建种子）
- 服务保活：WSL 会话退出会杀 mvn 进程，须用 Hermes terminal background 模式跑
  `wsl -d Ubuntu-24.04 -- bash -lc 'cd .../bulkhaul-server && exec mvn -q -o spring-boot:run'`

**差距评估（2026-08-27 初评，2026-08-29 闭合）**：
- flow.js 157 导出函数 vs 后端方法名比对：已实现 103 个（含 FlowCtx 内部 helper），54 个无同名方法
- **29 个写端点（真实缺口）已于 2026-08-29 全部实现并验证（verify-admin 51/51）**：
  - 管理后台 CRUD（AdminService）：saveCommodity / saveDriver / saveTerminal / saveWarehouse /
    toggleCommodityStatus / toggleCustomerStatus / toggleDriverStatus / importCommodities /
    importCustomers / importDrivers / importVehicles / sendVehicleRepair / resumeVehicle
  - 用户/角色/权限（UserAdminService，双存储 sys_user+biz_users）：saveUser / removeUser /
    toggleUserStatus / resetPassword / saveRole / removeRole / updateRolePerms / setDataScope /
    setDnd / dataScopeOf / getDnd / visibleMessages / unreadCount / markAllMessagesRead / markMessageRead
  - 运价卡（RateCardService）：createRateCard / updateRateCard / toggleRateCard / rateOf
  - 客户门户（ContractService）：submitTransportRequest / convertRequestToContract / rejectTransportRequest
  - 仓储（WarehouseService）：setSafetyStock / setInventoryStatus
  - 全局校准（RecalcService）：recalcAll
  - 端点统一挂在 `/api/admin/*`（客户门户在 `/api/contract/request/*`，仓储在 `/api/warehouse/*`）
- 14 个读取/纯计算：listRateCards / listInsuranceClaims 由 `/api/coll/*` 覆盖；rateOf / prepaymentOf /
  recalcAll / unreadCount / markAllMessagesRead / markMessageRead / dataScopeOf / getDnd /
  visibleMessages 已补端点；driverReportException / issueInvoiceRow 由现有 dispatch/结算端点覆盖；
  recordRegion / visibleDispatches / visiblePlans / inDataScope / isMuted 为前端本地过滤，无需后端
- 6 个前端本地会话 helper（clearOperator / setOperator）——纯 localStorage 逻辑，无需后端
- **阶段 6 剩余：前端 mock 层切真实 HTTP client（37 视图不动，仅换 src/mock 实现）**

## 六、环境现状（2026-08-26 实测，Ubuntu-24.04 WSL2）

- 资源：16 核 / 54GB 内存 / 945GB 可用磁盘（充足）
- 已装：Python 3.12、git、curl；systemd 正常运行（可 systemctl 管服务）
- 未装：Java/Maven/MySQL/Redis/Docker/Node
- 权限：zhuojun 已配免密 sudo（/etc/sudoers.d/zhuojun），可完成全部安装与服务管理
