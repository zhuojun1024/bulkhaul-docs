# 前端切真实 API 联调（阶段 6 收尾）

> 本文件是"前端 mock 层切真实 HTTP API"的**唯一事实来源**。上下文压缩后重读本文件 + 代码即可接上。
> 纪律：每步实现 → 真实工具输出验证（npm test / build / 浏览器冒烟 / 后端 verify 脚本）→ 才标 done-verified。
> 状态：pending / in-progress / done-verified / blocked

## 架构决策（2026-08-29 定稿）

**核心约束（实测确认）**：
1. `npm test`（verify-flow.mjs，554 断言）以**同步**方式调用 flow.js 函数（0 个 await）——它是内存引擎的规格，必须保持全绿。
2. 37 个视图**同步**调用写函数并立即用返回值（`if (guardError(flowConfirmLoad(row))) return`）。
3. 后端 service 返回与 flow.js **同构**（`{ok:true}` / `{error}`），外层包 `ApiResult{ok,data,error,code}`。
4. 后端**无 CORS** → 前端用相对 `/api` + 反向代理（dev server + verify-ui 静态服务）。
5. `db.logs` 被日志页读取，但后端审计在 `op_log` 表（不在 DataStore 33 集合内）→ 需 logs 端点。
6. `dashboard.js` 已是响应式（getter 读 db）→ 无需改。
7. 后端验证码 SVG 用 `<text>` 元素（verify-ui 可读），且回传 `code` 供自动化。

**方案 = 乐观本地 + 后台持久化 + 刷新（Option A）**：
- 浏览器中，写函数**同步跑内存引擎**（立即 UI 反馈 + 同步返回值，视图零改动），
  随后**后台**把同一操作 POST 到后端持久化，成功后从 `/api/snapshot` **刷新 db**（后端为权威态）。
- node 中（npm test）`USE_API=false`，纯内存，不发 HTTP → 554 断言不变。
- 读/计算函数（settlementCandidates/creditCheck/outstandingOf/visibleDispatches/…）**保持同步本地**，读 db（db 由后端 hydrate）。
- 启动时 `hydrate()`：GET /api/snapshot → 覆盖 db（读全部来自后端）。
- 内存引擎与后端引擎是同一逻辑（1:1 移植）、同一种子（V3 由前端 mock 生成）→ 分歧仅 RNG 字段（操作人/磅重/轨迹），刷新即收敛，无可见回退。

**端点映射**：后端 14 个 Controller 已覆盖全部 30+ 写端点 + /api/coll/* 读 + auth + scheduler。
新增 2 个：`GET /api/snapshot`（34 集合全量）+ `GET /api/logs`（op_log 审计）。

## 任务清单

### A1 [P0] 后端 /api/snapshot + /api/logs 端点 — done-verified
- 新增 `SnapshotController`：GET /api/snapshot → 34 集合（29 list + 5 object，DataStore）+ logs（op_log 最近 1000，时间倒序）。GET /api/logs 同。
- 需 JWT 认证（非 permitAll）。
- 文件：bulkhaul-server/src/main/java/com/blms/common/SnapshotController.java
- 验证：编译 EXIT=0 + 重启 + curl（带 token）返回 34 集合 + logs（commodities 16 / dispatches 204 / logs 118…）。
- 补：WeighingController 加 `POST /api/weighing/manual`（manualWeighing 原无端点，WeighingService 已有方法）。

### A2 [P0] 前端 src/mock/api.js — done-verified
- `USE_API`（typeof window !== 'undefined' 且非 node 测试态）。
- `api` HTTP client：fetch 相对 `/api`，自动带 `Authorization: Bearer <localStorage.blms_token>`，解析 ApiResult，401 → 清 token。
- `refreshDb()`：GET /api/snapshot → 覆盖 db 各集合（list 用 splice 保响应式，object 清除旧键+合并，logs 时间 ISO→空格归一）。
- `hydrate()`：启动时 refreshDb（带 token 时）。
- `afterWrite(fnName, ...args)`：按 W 映射（97 写函数，位置参数 args[i]）POST 后端 → 防抖 200ms refreshDb；失败 console.warn + refreshDb（回退权威态）。node 下 no-op。
- 文件：src/mock/api.js（第八轮已拆分为 `src/api/index.js`，见 fix-plan 第八轮）

### A3 [P0] flow.js 写函数挂 afterWrite — done-verified
- 97 个写函数入口加一行 `afterWrite('fnName', <位置参数>)`（node 下 no-op，浏览器下持久化+刷新）。
- 位置参数与函数签名一致（如 createDispatches(p,count,vehicleIds) → afterWrite('createDispatches',p,count,vehicleIds)）。
- login/generateCaptcha 不在此列（A4 登录视图直接走真实 API）。
- 4 个良性交叉调用（autoMatchBank→matchBankRecord / createContract→submitContractApproval / issueInvoiceRow→issueInvoice / matchBankRecord→recordPayment）各自映射独立端点，顺序保持。
- 文件：src/mock/flow.js（import { afterWrite } from './api' + 97 处入口插入）
- 验证：npm test **554 通过，0 失败**（内存引擎不变，afterWrite node 下 no-op）。

### A4 [P0] 登录/验证码切真实 API — done-verified
- login/index.vue：generateCaptcha → GET /api/auth/captcha（异步）；login → POST /api/auth/login（异步）。
- store/index.js：login(user, token) 存真实 JWT；userInfo 用后端返回的 user。
- main.js：启动时若 localStorage 有 token → hydrate() + setOperator（GET /api/auth/me）。
- 文件：src/views/login/index.vue、src/store/index.js、src/main.js
- 验证：curl 全链路 200（captcha→login→token→snapshot→logs→me→tick）；verify-ui 场景15 登录/失败审计/只读 RBAC 全绿。

### A5 [P0] scheduler 切真实 tick + 代理 — done-verified
- scheduler.js：runSchedulerTick → POST /api/scheduler/tick + refreshDb（USE_API 时）。
- vue.config.js：devServer.proxy `/api` → http://localhost:8081。
- verify-ui.mjs 静态服务：加 `/api` 反向代理到 8081。
- 文件：src/mock/scheduler.js、vue.config.js、scripts/verify-ui.mjs
- 验证：curl POST /api/scheduler/tick 200；verify-ui 场景13 在途进度自动推进（真实 tick）通过。

### A6 [P0] 全量验证 — done-verified
- npm test **554 通过，0 失败**（内存引擎不变）。
- npm run build **成功**。
- 浏览器 E2E：verify-ui.mjs **82/82 通过**（19 场景：登录/主链路/银行对账/客户导入/RBAC/在途推进/司机扫码/对账异议/质量扣减/安全库存/数据范围 等）。
- 后端 verify 脚本（fresh seed 有序跑）：**auth 23 + mainflow 44 + settlement 35 + aux 34 + phase45 57 + scheduler 10 = 203/203 全绿**。
- 文件：—

### A7 [P1] verify-ui 切后端权威态（去 localStorage 快照）— done-verified
- 旧架构 `db` 持久化到 `localStorage['blms_db_snapshot']`（persist.js enableAutoSave），新架构后端为权威（main.js 已移除 enableAutoSave，persist.js 成死代码）→ verify-ui 6 处 `page.waitForFunction` 读该 key 全部 15s 超时崩溃。
- 改法：verify-ui 加 node 侧 `getBackendSnapshot()`（lazy 登录 admin 取 token → GET /api/snapshot）+ `waitForBackend(condFn)` 轮询后端，替换全部 6 处 localStorage 读取（场景14 司机扫码 / 场景16 对账 / 场景17 质量扣减 / 场景17 dnd / 场景18 安全库存 / 场景18 数据范围）。
- `login()` 竞态修复：onLogin 在 `blms_user` 落盘后才 `hydrate()` 填 db.users 并 router.push(home)，路由守卫读 db.users 未 hydrate 会重定向 /login → 登录 helper 改为 `page.waitForFunction` 等浏览器**离开** `#/login`（post-hydrate + post-router.push）再返回。
- 状态漂移修复：共享持久后端被场景1-15 消费（CUS001 结算态被场景4/11 消耗）→ 后端加 `POST /api/admin/reset-demo`（DataStore 启动 captureSeed 深拷贝，resetToSeed 恢复内存，仅内存不回写 DB）；verify-ui 场景16/17/18/19 前 `await resetDemo()`，恢复"每场景=新鲜种子"语义。
- 文件：scripts/verify-ui.mjs、bulkhaul-server/src/main/java/com/blms/store/DataStore.java、bulkhaul-server/src/main/java/com/blms/common/SnapshotController.java

### A8 [P1] 种子时间脆弱性修复（fresh seed 长期有效）— done-verified
- 根因：V3 种子原为 2026-08-26 冻结（intransit eta 当时为未来），随时间推移 eta 变过去 → 3s tick 全转 exception → 漂移。重生种子用 `dayjs()` 取当前本地时间。
- 在途 eta 混合（src/mock/dispatch.js）：原 intransit eta **恒为未来**（`dayjs(NOW).add(...)`），导致 verify-scheduler 的"超 ETA 触发延误围栏"断言失败。改为按 `dSeq` 奇偶确定性分配：偶数序号 eta 已过（progress 70-95，触发围栏），奇数仍在途（eta 未来）。种子既含"延误"演示也含"正常在途"演示。
- verify-scheduler.mjs：`NOW` 原用 `toISOString()`（UTC），与种子 `dayjs()`（本地）不一致，00:00–08:00 本地时段 UTC 落后 8h 漏判已过 eta → 改本地时间。
- verify-phase45.mjs：`_d(0)` 原用 `toISOString().slice(0,10)`（UTC 日期），与后端 `ctx.today()`（本地）不一致，00:00–08:00 本地时段取到前一天 → "培训日期不能早于今天" → 改本地日期。
- 文件：src/mock/dispatch.js、scripts/V3__biz_tables.sql（+ 后端同步）、bulkhaul-server/scripts/verify-scheduler.mjs、bulkhaul-server/scripts/verify-phase45.mjs

## 压缩后重入协议
1. 重读本文件，找第一个非 done-verified 项；
2. 跑 `npm test` 判断 flow.js 现状（A3 分批验证的基准）；
3. 一切以本文件 + 代码现状为准，不凭摘要印象动手。
