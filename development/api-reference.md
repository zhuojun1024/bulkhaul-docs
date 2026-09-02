# API 参考（BulkHaul）

> 生成自前端写端点契约 src/api/endpoints.js（97 写端点）+ 读端点清单。前后端契约由 npm run test:contract 持续校验（method+path 与后端 Controller 路由一致）。
> 被测版本：前端 944c115 / 后端 ad0b06e（2026-09-02）。路径变量以 {id} 表示。

## 通用约定

| 项 | 说明 |
|---|---|
| Base URL | /api（前端经反向代理到后端 8081） |
| 鉴权 | Authorization: Bearer {JWT}；除 /auth/captcha、/auth/login、/health 外全部端点需登录 |
| 响应包装 | ApiResult：{ ok: boolean, data: T, error?: string, code?: string } |
| 错误码 | unauthenticated（401 未登录/过期）、conflict（409 乐观锁版本不匹配）、forbidden（RBAC 拒绝）、bad-request（参数/守卫拒绝）、not-found |
| 乐观锁 | 核心集合既有记录写可附 expectedVersion（body 或查询参数）；不匹配 → 409 conflict |
| 写后同步 | 前端写操作 POST 后防抖 200ms 调 GET /snapshot 重取权威态 |

## 读端点

| Method | Path | 说明 |
|---|---|---|
| GET | /health | 健康检查（db 连接 + 表数） |
| GET | /auth/captcha | 图形验证码（SVG，一次性，60s 有效） |
| POST | /auth/login | 登录（username/password/captchaId/captchaCode → JWT） |
| GET | /snapshot | 全量快照（34 集合 + logs），前端 db 水合/刷新源 |
| GET | /coll/{name} | 集合读（29 数组型集合，行级数据范围过滤） |
| GET | /dispatch/{id}/detail | 调度详情聚合（合同/计划/车辆/司机/磅单/异常/轨迹） |
| GET | /workbench/stats | 工作台统计（今日调度/在途/待办等） |
| GET | /workbench/todos | 工作台待办列表 |
| GET | /dashboard/kpi | 看板 KPI 指标 |
| GET | /dashboard/charts | 看板图表数据 |
| POST | /scheduler/tick | 定时任务驱动（验收栈手动驱动；生产栈由后端调度器自动调用） |

## 写端点（97）

### 调度（/api/dispatch）

| 操作 | Method | Path | Body |
|---|---|---|---|
| confirmLoad | POST | /dispatch/{id}/confirmLoad | 无 |
| depart | POST | /dispatch/{id}/depart | 无 |
| arrive | POST | /dispatch/{id}/arrive | 无 |
| confirmUnload | POST | /dispatch/{id}/confirmUnload | 无 |
| cancelDispatch | POST | /dispatch/{id}/cancel | 有 |
| reassignDispatch | POST | /dispatch/{id}/reassign | 有 |
| reportException | POST | /dispatch/{id}/reportException | 有 |
| driverReportException | POST | /dispatch/{id}/reportException | 有 |
| resumeDispatch | POST | /dispatch/{id}/resume | 无 |
| createDispatches | POST | /dispatch/create | 有 |
| acceptDispatch | POST | /dispatch/{id}/accept | 无 |
| driverDepart | POST | /dispatch/{id}/driver/depart | 无 |
| driverArrive | POST | /dispatch/{id}/driver/arrive | 无 |
| signReceipt | POST | /dispatch/{id}/driver/signReceipt | 有 |
| supplementReceipt | POST | /dispatch/{id}/supplementReceipt | 有 |
| scanConfirmLoad | POST | /dispatch/{id}/scan/load | 有 |
| scanConfirmUnload | POST | /dispatch/{id}/scan/unload | 有 |

### 磅单（/api/weighing）

| 操作 | Method | Path | Body |
|---|---|---|---|
| manualWeighing | POST | /weighing/manual | 有 |
| correctWeighing | POST | /weighing/{id}/correct | 有 |

### 仓储（/api/warehouse）

| 操作 | Method | Path | Body |
|---|---|---|---|
| manualInbound | POST | /warehouse/inbound | 有 |
| setSafetyStock | POST | /warehouse/safetyStock | 有 |
| setInventoryStatus | POST | /warehouse/inventory/{id}/status | 有 |

### 异常（/api/exception）

| 操作 | Method | Path | Body |
|---|---|---|---|
| acceptException | POST | /exception/{id}/accept | 有 |
| finishException | POST | /exception/{id}/finish | 有 |
| closeException | POST | /exception/{id}/close | 无 |

### 安全（/api/safety）

| 操作 | Method | Path | Body |
|---|---|---|---|
| registerAccident | POST | /safety/accident | 有 |
| closeAccident | POST | /safety/accident/{id}/close | 无 |
| addTraining | POST | /safety/training | 有 |
| completeTraining | POST | /safety/training/{id}/complete | 有 |
| addInspection | POST | /safety/inspection | 有 |

### 保险（/api/insurance）

| 操作 | Method | Path | Body |
|---|---|---|---|
| fileInsuranceClaim | POST | /insurance/claim | 有 |
| assessInsuranceClaim | POST | /insurance/claim/{id}/assess | 有 |
| settleInsuranceClaim | POST | /insurance/claim/{id}/settle | 有 |
| rejectInsuranceClaim | POST | /insurance/claim/{id}/reject | 有 |

### 结算（/api/settlement）

| 操作 | Method | Path | Body |
|---|---|---|---|
| generateSettlements | POST | /settlement/generate | 有 |
| startReconcile | POST | /settlement/{id}/startReconcile | 无 |
| recalcSettlement | POST | /settlement/{id}/recalc | 无 |
| confirmSettle | POST | /settlement/{id}/confirmSettle | 无 |
| recordPayment | POST | /settlement/{id}/recordPayment | 有 |
| revertPayment | POST | /settlement/{id}/revertPayment/{id} | 有 |
| dunning | POST | /settlement/{id}/dunning | 有 |
| customerConfirm | POST | /settlement/{id}/customerConfirm | 无 |
| customerObjection | POST | /settlement/{id}/customerObjection | 有 |
| applyPrepayment | POST | /settlement/{id}/applyPrepayment | 有 |
| collectPrepayment | POST | /settlement/prepayment/collect | 有 |
| issueInvoice | POST | /settlement/{id}/issueInvoice | 无 |
| issueInvoiceRow | POST | /settlement/{id}/issueInvoice | 无 |
| redFlushInvoiceRow | POST | /settlement/invoice/{id}/redFlush | 有 |

### 财务核销（/api/finance）

| 操作 | Method | Path | Body |
|---|---|---|---|
| generatePayables | POST | /finance/payables/generate | 无 |
| payPayable | POST | /finance/payables/{id}/pay | 有 |
| addBankStatement | POST | /finance/bank/statement | 有 |
| matchBankRecord | POST | /finance/bank/{id}/match | 有 |
| autoMatchBank | POST | /finance/bank/autoMatch | 无 |
| createContract | POST | /contract | 有 |
| createPlan | POST | /plan | 有 |
| cancelPlan | POST | /plan/{id}/cancel | 无 |
| submitContractApproval | POST | /contract/{id}/submitApproval | 无 |
| approveContract | POST | /contract/{id}/approve | 有 |
| rejectContract | POST | /contract/{id}/reject | 有 |
| changeContract | POST | /contract/{id}/change | 有 |
| approveContractChange | POST | /contract/{id}/approveChange | 有 |
| rejectContractChange | POST | /contract/{id}/rejectChange | 有 |
| extendContract | POST | /contract/{id}/extend | 有 |
| terminateContract | POST | /contract/{id}/terminate | 有 |
| completeContract | POST | /contract/{id}/complete | 无 |
| archiveContract | POST | /contract/{id}/archive | 无 |
| submitTransportRequest | POST | /contract/request | 有 |
| convertRequestToContract | POST | /contract/request/{id}/convert | 有 |
| rejectTransportRequest | POST | /contract/request/{id}/reject | 有 |

### 管理后台（/api/admin）

| 操作 | Method | Path | Body |
|---|---|---|---|
| saveCommodity | POST | /admin/commodity | 有 |
| toggleCommodityStatus | POST | /admin/commodity/{id}/toggle | 无 |
| saveTerminal | POST | /admin/terminal | 有 |
| saveWarehouse | POST | /admin/warehouse | 有 |
| saveDriver | POST | /admin/driver | 有 |
| toggleDriverStatus | POST | /admin/driver/{id}/toggle | 无 |
| toggleCustomerStatus | POST | /admin/customer/{id}/toggle | 无 |
| importCommodities | POST | /admin/commodity/import | 有 |
| importCustomers | POST | /admin/customer/import | 有 |
| importDrivers | POST | /admin/driver/import | 有 |
| importVehicles | POST | /admin/vehicle/import | 有 |
| sendVehicleRepair | POST | /admin/vehicle/{id}/repair | 有 |
| resumeVehicle | POST | /admin/vehicle/{id}/resume | 无 |
| saveUser | POST | /admin/user | 有 |
| removeUser | DELETE | /admin/user/{id} | 无 |
| toggleUserStatus | POST | /admin/user/{id}/toggle | 有 |
| resetPassword | POST | /admin/user/{id}/resetPassword | 有 |
| saveRole | POST | /admin/role | 有 |
| removeRole | DELETE | /admin/role/{id} | 无 |
| updateRolePerms | PUT | /admin/role/{id}/perms | 有 |
| setDataScope | PUT | /admin/user/{id}/dataScope | 有 |
| setDnd | PUT | /admin/dnd | 有 |
| createRateCard | POST | /admin/rateCard | 有 |
| updateRateCard | PUT | /admin/rateCard/{id} | 有 |
| toggleRateCard | POST | /admin/rateCard/{id}/toggle | 无 |
| recalcAll | POST | /admin/recalc | 无 |

### 消息（/api/admin）

| 操作 | Method | Path | Body |
|---|---|---|---|
| markMessageRead | POST | /admin/messages/{id}/read | 无 |
| markAllMessagesRead | POST | /admin/messages/readAll | 无 |

## 集合清单（/coll/{name} 与 /snapshot 覆盖）

数组型（29）：commodities, customers, terminals, vehicles, drivers, contracts, transportRequests, plans, dispatches, weighings, warehouses, inventories, settlements, payments, prepayments, payables, dunnings, bankRecords, invoices, messages, exceptions, accidents, trainings, inspections, rateCards, insurance, safetyStocks, users, roles

对象型（5）：rolePerms, fenceConfig, escalateConfig, dnd, dataScopes
