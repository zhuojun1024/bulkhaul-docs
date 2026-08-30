# 修复进度计划（fix-plan）

> 本文件是修复工作的**唯一事实来源**。上下文压缩后，重读本文件 + 代码即可接上进度。
> 纪律：每项先实现 → 补 verify-flow 断言 → `npm test` 全绿 → 才标记 done-verified。
> 状态：pending / in-progress / done-verified / blocked

## 基线（2026-08-25 开工前）

- `npm test`：486 通过，0 失败（verify-flow 环节 1–25）
- 新增断言统一从环节 26 起编号

## 任务清单

### F1 [P0] 卸货中异常恢复分支 — done-verified（环节 26，8 断言）
- 问题：`resumeDispatch`（flow.js:830）只按 loadTime 二分恢复（intransit/loading），
  异常发生在"卸货中"时恢复回 intransit，需重走 到达→卸货，且可能二次过磅。
- 方案：`createException` 记录 `d.exceptionFrom = d.status`（异常前状态）；
  `resumeDispatch` 优先按 exceptionFrom 恢复（loading/intransit/unloading 各自回原态），
  无 exceptionFrom 时回退现有 loadTime 二分逻辑（兼容种子数据）。
  unloading 恢复：progress=96，eta 顺延，不重算 speed（卸货中无车速）。
- 文件：src/mock/flow.js（createException ~777、resumeDispatch ~830）
- 测试：verify-flow 环节 26（unloading 异常→恢复回 unloading→可继续 confirmUnload；
  intransit/loading 恢复行为不变；种子兼容回退）
- 验收：npm test 全绿

### F2 [P0] 仓储手工入库/补库 — done-verified（环节 27，9 断言）
- 问题：装货出库有充足性守卫，但全系统无手工入库入口，库存扣到 0 后装货被拦截且无法补库（死路）。
- 方案：`manualInbound(warehouseId, commodityId, quantity, batch?, remark?)`（RBAC: warehouse）
  - 守卫：仓库存在且 operating、商品存在、quantity>0；
  - 生成批次（batch 缺省 `B{YYMMDD}-M{seq}`），status=normal，inDate=今天；
  - wh.used 累加（不超 capacity，超出报错）；留痕 + 通知仓储角色。
- 文件：src/mock/flow.js（warehouseIn 附近）、src/views/warehouse/inventory.vue（入库按钮 + 对话框）
- 测试：verify-flow 环节 27（正常入库/守卫拦截/used 累加/超容量拦截）
- 验收：npm test 全绿

### F3 [P1] 门户开放对账明细 — done-verified（纯 UI，build 验证）
- 问题：客户门户只展示"差异 N 车次、损耗 X 吨"汇总，看不到逐车次明细，确认/异议缺乏依据。
- 方案：portal/index.vue 账单表格加展开行（el-table type=expand），逐车次展示
  调度单号/车牌/调度量/进磅/出磅/结算量/损耗/质量扣重/差异/签收/比对，
  数据取 s.reconciliation.items；未对账账单显示空态。
- 文件：src/views/portal/index.vue
- 测试：纯 UI 展示，无服务层变更（verify-ui 已有门户场景）
- 验收：build 通过

### F4a [P1] 司机端异常上报 — done-verified（环节 28，5 断言）
- 问题：司机是事故/故障第一知情人，driver/app.vue 无 reportException 入口。
- 方案：`driverReportException(d, description, type, level)`（走 requireDriverApp 身份守卫 +
  createException 内部核心，source='driver'）；H5 车次卡片执行中状态显示"上报异常"按钮 + 对话框。
- 文件：src/mock/flow.js（司机端区块）、src/views/driver/app.vue
- 测试：verify-flow 环节 28（司机本人上报成功/非本人司机拦截/非司机角色拦截/事故类联动）
- 验收：npm test 全绿

### F4b [P1] 司机新增/导入 — done-verified（环节 29，11 断言）
- 问题：司机只有种子数据，无新增表单、无导入，人员变动无法维护。
- 方案：`saveDriver(payload)`（RBAC: driver，新建/编辑，手机号查重）+
  `importDrivers(rows)`（RBAC: driver，按手机号去重，与 importVehicles 同模式）；
  driver/list.vue 加"新增司机"按钮 + 表单对话框 + "导入"按钮（复用 ImportDialog）+ 行内"编辑"。
- 文件：src/mock/flow.js（写操作下沉区块）、src/views/driver/list.vue
- 测试：verify-flow 环节 29（新建/必填拦截/查重/编辑/导入去重/RBAC）
  注意：调度员本身持有 driver 权限，RBAC 反例用结算专员。
- 验收：npm test 全绿

### F4c [P1] 管理员重置密码 — done-verified（环节 30，9 断言）
- 问题：saveUser 编辑不能改密码，用户忘记密码/锁定后无恢复入口。
- 方案：`resetPassword(userId, newPassword)`（RBAC: user）：校验密码非空且 ≥6 位，
  更新 passwordHash，审计留痕（不落新密码明文），通知目标用户角色（停用跳过）。
  system/user.vue 加"重置密码"按钮 + 对话框（二次确认输入）。
- 文件：src/mock/flow.js（saveUser 附近）、src/views/system/user.vue
- 测试：verify-flow 环节 30（重置后旧密码失败/新密码成功/短密码拦截/RBAC/审计留痕）
  注意：审计日志在 db.logs，字段为 action/detail。
- 验收：npm test 全绿

### F5a [P2] 报表运量口径统一 — done-verified（环节 31，3 断言）
- 问题：monthlyReport/customerReport/commodityReport 运量仍用调度量 d.quantity，
  与结算（出磅净重）口径不一致。
- 方案：三报表运量改用 settleQtyOf 口径（出磅净重，无出磅单回退调度量）。
  settleQtyOf 由 flow.js 私有改为导出。report.js 头部口径说明同步更新。
- 文件：src/mock/flow.js（导出 settleQtyOf）、src/mock/report.js
- 测试：verify-flow 环节 31（构造调度量100/出磅98的车次，三报表对应行各 +98 非 +100）
- 验收：npm test 全绿

### F5b [P2] 计划拆车余数修正 — done-verified（环节 32，3 断言）
- 问题：createDispatches 的 per=round(quantity/count) 均摊，100 吨拆 3 车 = 34×3=102 吨超量。
- 方案：每车至少 1 吨，实际拆车数 effCount = min(count, floor(quantity))；
  前 effCount-1 车取 floor 均摊 per，最后一车取余数（quantity - per*(effCount-1)），
  保证 Σ车次量 = 计划量。公路/非公路两分支同改（含 fee 按实际量计算）。
- 文件：src/mock/flow.js（createDispatches）
- 测试：verify-flow 环节 32（100吨拆3车=33/33/34；90吨整除=30/30/30 不变；
  2吨拆3车降级为2车各1吨）
- 验收：npm test 全绿

### FINAL 收尾 — done-verified
- npm test 全绿（环节 1–32，533 断言，0 失败）
- npm run build 成功（vue-cli-service build，dist 可部署）
- 已更新本文件全部状态 + README（辅助流程/功能模块/测试与质量章节）

---

## 第二轮整改（基础数据维护 + 财务核销链路）

> 触发：重新评估"功能完整性/流程闭环/逻辑自洽"后，识别出基础数据维护（场站/仓库）
> 与财务核销（银行流水来源）两类缺口。基线 533 → 目标 554+。

### F6a [P1] 场站新增/编辑 — done-verified（环节 33，8 断言）
- 问题：terminal/list.vue 只读，flow.js 无 saveTerminal；合同强依赖装/卸场站，
  但场站只能靠种子数据，与已补齐的司机/车辆/商品/客户不对称。
- 方案：`saveTerminal(payload)`（RBAC: terminal）：名称必填且查重、类型合法
  （loading/unloading/both）、日能力>0；新建默认 operating/吞吐0/无配套仓库；
  编辑可改配套仓库（影响装卸货仓储联动）。permission-table 加 `terminal` 操作码
  （授场站操作员）。terminal/list.vue 加"新增场站"按钮 + 卡片"编辑" + 表单对话框。
- 文件：src/mock/flow.js（saveTerminal）、src/permission-table.js、src/views/terminal/list.vue
- 测试：verify-flow 环节 33（新建/结构/名称缺失/重名/能力非正/编辑/调度员越权拦截）
- 验收：npm test 全绿

### F6b [P1] 仓库新增/编辑 — done-verified（环节 34，8 断言）
- 问题：warehouse/list.vue 只读，flow.js 无 saveWarehouse；仓库只能靠种子数据。
- 方案：`saveWarehouse(payload)`（RBAC: warehouse-maint）：名称必填且查重、容量>0；
  新建默认 operating/used=0；编辑容量不得低于已用库存（used）。permission-table 加
  `warehouse-maint` 操作码（授场站操作员）。warehouse/list.vue 加"新增仓库"按钮 +
  卡片"编辑" + 表单对话框。
- 文件：src/mock/flow.js（saveWarehouse）、src/permission-table.js、src/views/warehouse/list.vue
- 测试：verify-flow 环节 34（新建/结构/名称缺失/重名/容量非正/容量下限守卫/编辑/越权拦截）
- 验收：npm test 全绿

### F7 [P1] 银行流水录入 — done-verified（环节 35，7 断言）
- 问题：db.bankRecords 只有种子数据，全代码无录入入口；核销流程（手动/自动）逻辑
  完整但"流水从哪来"断链，演示时只能核销种子流水。
- 方案：`addBankStatement(payload)`（RBAC: settlement）：对手方/金额/到账时间必填、
  金额>0；录入后 status=unmatched 进入待核销，可走自动核销（金额=账单未付余额）
  或手动核销，闭合"流水从哪来"。settlement/list.vue 银行对账页加"流水录入"按钮 +
  对话框（对手方下拉可输入/金额/到账时间/摘要）。
- 文件：src/mock/flow.js（addBankStatement）、src/views/settlement/list.vue
- 测试：verify-flow 环节 35（对手方缺失/金额非正/时间缺失/录入成功 unmatched/结构/越权拦截）
- 验收：npm test 全绿

### F8 [P2] 已知取舍（留档 backlog，不实现）
- 说明：以下三项在演示系统层面可接受，作为"已知取舍"留档，暂不实现。
  若走向真实系统再补。
  1. 异常损失单一金额字段：finishException 的 cost 不区分货损/车损/维修费，
     真实理赔责任认定需拆分；演示用单字段+备注可接受。
  2. 客户门户不能自助预付：collectPrepayment 仅平台侧（客户详情页）操作，
     真实系统客户应能自助充值；演示由销售代操作可接受。
  3. 银行自动核销仅精确匹配：一笔流水只核销一张账单、容差 0.01 元，
     真实银行常"一笔付多账/含手续费"会落到人工核销；演示可接受，UI 已说明口径。

### FINAL-2 收尾 — done-verified
- npm test 全绿（环节 1–35，554 断言，0 失败）
- npm run build 成功
- 已更新本文件 + README（场站/仓库/银行流水录入）
- 已 commit + push（9f040b9，8 文件 +576/-26）

---

## 第三轮优化（全局 UI：操作列自适应宽度）

### F9 [P1] 操作列按可见按钮数自适应宽度 — done-verified（verify-actioncol E2E，5 断言）
- 问题：带操作列的表格按钮受 RBAC + 行状态双重控制，数量动态；列宽写死导致
  权限少的账号（如调度管理只见"详情"）看到过宽空白列（300px）。
- 方案：新增全局组件 `src/components/ActionColumn.vue`——包装 el-table-column，
  列宽绑定响应式值；每行单元格用 MutationObserver 监听按钮 v-if 增删，
  列宽 = clamp(当前页最宽行按钮总宽 + 16, minWidth 80, maxWidth 420)。
  20 个视图文件 28 处操作列批量替换（含 v-if 列常显的保留原 v-if）。
- 顺带修复：dispatch/list.vue "改派"按钮绑定 `reassign(row)` 但函数名为
  `openReassign`（未定义，点击抛错、对话框打不开）→ 函数改名 reassign；
  portal/index.vue 6 处全角空格（历史 lint 错误）→ 普通空格。
- 测试：scripts/verify-actioncol.mjs（puppeteer E2E：只读用户筛"待装货"操作列 80px
  vs 管理员同筛选 250px，随按钮数收敛；npm test 554 全绿、build 干净、
  verify-ui 19 组场景回归）
- 验收：列宽随权限/行状态动态收敛，无固定宽空白列

## 第四轮：测试红转绿（种子漂移根治）— done-verified [53426b4]

- 现象：文档全标 done-verified 但 `npm test` 实为红——M3 崩溃（m3batches 空）
  掩盖 N-2（4 项）+ 环节13（4 项）共 8 个失败。
- 根因：WH007×CM001 批次状态由全局 rng 随机生成；M3 测试写入（060833b）之后，
  e35c38b（rate/weighing）、4dc28c1（dispatch）新增 rng() 消耗，种子序列漂移，
  WH007 的 CM001 可发（normal）库存变为 0 → M3 出库守卫拦截种子计划 YH-0019
  等装货车次，N-2/环节13 状态机断言连锁失败。
- 修复（不消耗 rng，不扰动下游种子）：
  1. warehouse.js：确定性垫批 INV-SEEDWH7——WH007×CM001 normal 可发 < 2000 吨时
     补足（同环节7 垫批口径），保证装货闭环测试不依赖随机种子状态；
  2. M3 测试：自造受控批次基础上临时锁定种子 normal 批次（含垫批），
     "库存不足"子测试口径完全受控，测完恢复；
  3. 环节13：confirmSettle 要求结算组内全部公路车次有签收单，测试只签了
     q13d 一个 → 组内其余公路车次补签后再结算（此前静默失败）。
- 验证：npm test 556 通过 0 失败；UI E2E（test:ui）在"银行对账自动核销"
  10s 超时——stash 掉本次改动后干净 HEAD 同样失败，属存量问题（后端 hydrate
  架构下种子/后端状态相关），与本次修复无关，未处理。

## 第五轮：UI E2E 存量失败根治（种子污染 + 时间敏感断言）— done-verified

- 现象：test:ui 场景11"银行对账自动核销"10s 超时（第四轮记录为存量问题）。
- 根因1（DB 种子污染）：后端 commitAll 把业务写操作回写 biz_*（前次 UI 测试自动核销
  成功写入 SK-0021 收款 → JS-0008 变 settled），重启后 DataStore 从污染态 DB 加载并
  captureSeed 捕获污染态 → reset-demo 也回不到正确种子 → autoMatchBank 永远匹配不到
  （JS-0008 未付=0、YH-0008 无账单可匹配）→ 超时。
- 修复1（bulkhaul-server）：
  1. V4__seed_snapshot.sql：把 biz_* 种子态固化到只读 seed_* 表（34 张）；
  2. DataStore.tryLoadSeedFromSnapshot：种子基线优先从 seed_* 加载（不受 commitAll
     污染影响），seed_* 缺失时回退内存捕获（旧库兼容）；
  3. 重放 V3 SQL 恢复 DB 种子态（JS-0008 回 overdue、SK-0021 清除）。
- 根因2（场景11 修复后暴露 3 个被掩盖的失败）：
  - P2 监控页（2 项）：种子在途车次 eta 是 dump 时刻的固定过去时间 → 前端 isDelayed
    全为延误（tag≠"在途"过滤不到）；且每次 scheduler tick 围栏 delay 分支命中 →
    createException 把车次转 exception；后端 auto-enabled=false 不自动 tick，
    依赖浏览器前端 timer（headless 下脆弱）。种子时间敏感，无法稳定复现。
  - 环节4（1 项）：断言写死"CUS001 已结算/逾期未付清账单"，种子漂移后 CUS001 只有
    reconciling/pending（当前种子 CUS003 才有预付+逾期未付）。
- 修复2（bulkhaul-manage-web）：
  1. verify-ui.mjs 脚本开头 + 场景13 开头 resetDemo()（等价旧架构"每场景全新种子"，
     防 commitAll 污染 + 场景 1-12 消耗种子）；
  2. 环节4/5 断言改动态定位"有预付款 + 有已结算/逾期未付清账单"的客户（不写死 CUS001）；
  3. P2 重写：动态创建新在途车次（confirmLoad+depart，eta=未来时间围栏不命中），
     node 侧手动驱动 /api/scheduler/tick（确定性，等价全局定时任务；auto-enabled=false
     下后端不自动 tick），后端快照 float 比较观察进度推进（UI 取整显示会 flaky）；
     track/index.vue 列表项加 data-dispatch-id（铁路车次 vehicleId=null 时 plate 映射不可靠）。
- 验证：npm test 556 通过 0 失败；test:ui 82 通过 0 失败，且**连续两次全绿**
  （幂等性：reset-demo 回种子态 + seed_* 快照防污染，跑多少次都稳定）。
- 遗留：src/mock/api.js:56 存量 lint error（no-empty Empty block statement）已修复
  （catch 块补注释，零行为变化，npx eslint src/mock/api.js 清零）；
  vue.config.js devServer port:8087 本地改动已还原（工作区与 HEAD 一致）。
- 经验沉淀：lessons-learned.md（环境拓扑/后端启动/验证基线/种子治理/断言纪律，同目录）；
  一次性脚本清理（server q-*/probe、web smoke-server、V1/V2 冗余 SQL 副本、dump-schema.json）。

## 第六轮：后端集成测试整改（mvn test 4 条失败断言 → 全绿）— done-verified

- 现象：bulkhaul-server `mvn test`（FlowIntegrationTest，环节 1-10 共享内存态、operator 注入平台管理员）
  4 条断言失败：环节8 状态机守卫（d2 应为 intransit）、环节9 异常关闭补扣、环节9 重算幂等、
  环节9 装卸码确定性派生。前端 mock（src/mock/flow.js）为业务权威口径，后端 1:1 平移时 3 处行为漂移 + 1 处类型错误。
- 根因与修复（均在 bulkhaul-server）：
  1. **装卸码（hashStr 类型错误）**：前端 `hashStr` 用 JS number（64 位 double，`n*31` 不溢出），
     后端误用 `int` → 溢出回绕产生负值 → `loadCodeOf` 码值 <100000 或负数，`ZD\d{6}` 格式/确定性破坏。
     修复：`DispatchService.hashStr` 改 `long`（`n*31 + ch) % 2147483647L`）。
  2. **createDispatches 公路分支插入顺序**：前端 `db.dispatches.unshift(d)` 正向 → 列表"最新在前"
     `[c2,c1,c0,...seed]`；后端误用反向 `for(i=last..0) add(0, pending[i])` → "最旧在前" `[c0,c1,c2,...seed]`。
     环节6 事故测试 `find(intransit+loadTime)` 取"第一个"，后端因此选中 d2（created[1]）置 exception 且不再
     恢复 → 环节8 守卫发现 d2=exception 失败（前端选中 created[2]，d2 保持 intransit）。修复：改正向
     `for(d of pending) add(0,d)` 对齐前端 unshift。
  3. **reportException 返回契约**：前端 `reportException` 直接返回异常单 `e`（`createException` 返回值）；
     后端复制了 createException 全部逻辑却返回 `{ok, exception: e}`（包装）→ 测试 `e13.id` 为 null，
     `finishException(null)`/`closeException(null)` 全落空 → 环节9 补扣/幂等连锁失败（重算幂等断言的
     totalAmount 子句是补扣未执行的级联）。修复：`DispatchService.reportException` 委托
     `ExceptionService.createException`（RBAC 单点校验 exception 后），直接返回 `e`；删除重复实现（消除双份漂移源）。
- 测试侧（FlowIntegrationTest）：加 `@BeforeAll resetToSeed()`（等价前端 verify-ui 的 resetDemo，防上次运行
  commitAll 污染 → 本次加载污染态 → 内存重复 ID → 主键冲突 → 连锁 NPE）；4 条断言补诊断信息（失败时打印
  状态/金额/码值，便于定位）。
- 验证：WSL Ubuntu-24.04（JDK17 / Maven 3.8.7 / MySQL blms_test / Redis）`mvn test` EXIT=0，
  环节 1-10 汇总 **PASS=141 FAIL=0**（含原 4 条失败断言）；前端 `npm test` 556 通过 0 失败（无回归）。
- 提交：bulkhaul-server（DispatchService.java + FlowIntegrationTest.java）；本文件同步（前端仓库）。

## 第七轮：整体代码扫描（三层交叉核对 + 潜在问题排查）— done-verified

- 范围：前端 mock 服务层（flow.js 95 导出函数）× api.js W 映射（97 写端点）× 后端 15 个 Controller（118 路由）
  三层交叉核对 + RBAC 覆盖 + 类型/契约漂移 + 确定性 + 健壮性。
- 核对结论（无问题）：
  - **端点/字段漂移**：97 个 W 端点全部命中后端路由（method+path 一致）；带 body 的 25 个端点逐字段比对
    （planId/count/vehicleIds、reason、description/type/level、handler、result/cost、amount/method、
    newNet/reason、status、fields/reason、newDate、settleNow、customerId、regions、password 等）
    与 Controller `body.get(...)` 全部一致，无静默丢字段。
  - **afterWrite 覆盖**：flow.js 55 个 afterWrite 写函数全部在 W 映射中（浏览器态持久化无遗漏）；
    其余 mock 模块无 afterWrite。
  - **RBAC 覆盖**：全部状态变更 service 方法均有 `ctx.requireAction` 单点校验；司机端 4 方法
    （acceptDispatch/driverDepart/driverArrive/signReceipt）用 requireDriverApp 身份守卫（与前端一致）；
    自助类（setDnd/markMessageRead/markAllMessagesRead）前后端均无权限守卫（本人数据，口径一致）；
    系统任务（scheduler tick/fence/createException/recalcSettlementStatus）无守卫（系统身份，口径一致）。
  - **集合清单**：前端 api.js LIST_COLLS(29)+OBJ_COLLS(5) 与后端 DataStore 完全一致。
  - **确定性**：genInvoiceNo 已用 long（无 hashStr 同款溢出）；Math.random 仅存在于 scheduler tick
    （运行时遥测推进，非派生码路径，可接受）；无 TODO/FIXME 残留标记。
  - **测试可信度**：FlowIntegrationTest @AfterAll `assertEquals(0, fail)` 真失败即构建红（非仅打印）。
- 修复（2 处，均在 bulkhaul-server）：
  1. **RecalcService.recalcAll 缺 RBAC 守卫**：POST /api/admin/recalc 全局口径校准（改写调度方式/
     车辆司机占用/计划合同进度/结算状态 + commitAll）无任何权限校验——SecurityConfig 仅要求已认证，
     且 rolePerms 种子中无任何角色授予 recalc/admin 操作 → 只读用户/客户/司机等任意登录角色可触发
     全局数据变更。修复：service 入口加 `ctx.requireAction("admin")`（仅平台管理员 actions=null 全放行）。
     前端启动时 recalcAll() 为内存本地校准，afterWrite 后台 POST 被拒仅 console.warn 不阻塞 UI（口径不变）。
  2. **DispatchController /{id}/codes NPE**：调度单不存在时 `dispatchOf(id)` 返回 null →
     `loadCodeOf(null)` → `str(null, "id")` NPE → 500。修复：null 时返回 ApiResult.fail（404 语义）。
- 验证：WSL `mvn test` EXIT=0，PASS=141 FAIL=0（两处修复无回归）；前端 eslint 0 错、npm test 556 全绿、
  npm run build 干净（扫描期间复验）。
- 遗留（不阻塞，记录备查）：
  - 浏览器态 afterWrite→后端 POST 链路无自动化契约测试（verify-ui 只覆盖 UI 行为）；字段漂移目前靠
    人工核对（本轮已全量核对一遍）。后续可加"W 映射 vs Controller 路由"的 CI 静态检查防回归。
  - DispatchController /probe 为阶段 2 鉴权探针（verify-auth.mjs 已随脚本清理删除，端点保留且带
    @RequireAction，无风险；如需可删）。

## 第八轮：目录拆分 src/mock/api.js → src/api/（接口层与内存引擎分离）— done-verified

- 背景：接口已全量对接（登录/快照/写持久化/定时任务走真实后端），但联调层仍留在 src/mock/ 下，
  目录语义与"mock 假数据"直觉冲突。拆分为常规前端项目结构：
  - `src/api/index.js`：真实 API 联调层（USE_API 开关 + api() HTTP client + refreshDb/hydrate + afterWrite + W 映射 97 端点）
  - `src/mock/`：内存业务引擎（flow.js 95 业务函数 + 种子数据 base/commodity/... + scheduler/persist/dashboard）
- 改动（纯移动 + 路径更新，零逻辑变化）：
  1. `git mv src/mock/api.js src/api/index.js`（保留 git 历史）；文件内 `./base` → `../mock/base`
  2. 引用点 5 处：flow.js `./api`→`@/api`、scheduler.js `./api`→`@/api`、main.js `./mock/api`→`./api`、
     views/login/index.vue `@/mock/api`→`@/api`
  3. `scripts/alias-loader.mjs`（node 测试 @/ 别名解析）：候选顺序改为文件优先（`.js` → `/index.js` → 裸路径），
     修复 `@/api` 命中目录导致 EISDIR 的问题（裸路径 existsSync 对目录为 true）
- 依赖方向：mock 引擎 → api 层（单向，api 只依赖 mock/base 的 db）；视图/store 直接 import `@/api`。
- 验证：eslint 0 错；npm test 556 通过 0 失败（node 纯内存态不受影响）；npm run build 干净；verify-ui 82 断言（浏览器真实 API 态）。
- 遗留：api-integration-plan.md 等历史文档中 `src/mock/api.js` 字样为当时记录，不回改（项目级文档已上移至 bulkhaul/docs/，与本文件同目录）。

## 压缩后重入协议

1. 重读本文件，找到第一个非 done-verified 的项；
2. 读 todo() 确认当前 in_progress；
3. 若某项处于 in-progress（改了代码未验证），先跑 `npm test` 判断现状再决定继续或回退；
4. 不凭摘要印象动手，一切以本文件 + 代码现状为准。
