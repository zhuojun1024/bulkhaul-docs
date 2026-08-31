# 验证与运维经验（lessons-learned）

> 2026-08-30 第五轮全量验证通过后沉淀。覆盖：环境拓扑、后端启动、全量验证跑法、
> 种子污染治理、时间敏感断言教训。上下文压缩后重读本文件可快速恢复验证能力。

## 1. 环境拓扑（Windows + WSL 分工）

| 组件 | 位置 | 说明 |
|---|---|---|
| bulkhaul-manage-web（前端 + 测试） | Windows 工作区 | npm test / build / E2E 都在 Windows 侧跑 |
| bulkhaul-server（Spring Boot 后端） | 源码在 Windows，**运行在 WSL** | Windows 无 Java/Maven，不能在本机起后端 |
| JDK 17 / Maven 3.8.7 / MySQL / Redis | WSL Ubuntu-24.04 | `systemctl is-active mysql redis-server` 均 active |
| 数据库 | WSL MySQL，`blms` 库（另有 blms_test） | 账号 blms/blms123456；105 张表 = biz_*(34) + seed_*(34) + sys_* 等 |
| WSL 访问 Windows 文件 | `/mnt/d/...` | 后端 cwd 用 /mnt/d 路径即可（run.sh 已配好） |

**端口分配（避免撞车）**：
- 8080 = llama-server（本机 LLM，**勿占用/勿依赖**）
- 8081 = bulkhaul-server（application.yml 固定）
- 8086 = verify-ui.mjs 静态服务（dist + /api 反代 8081）
- 8087 = vue devServer 本地联调（vue.config.js 未提交改动，保留）

## 2. 后端启动（WSL 内保活）

```bash
# WSL 内（等价 scripts/run.sh，run.sh 用 mvn -o 离线模式）
cd /mnt/d/Documents/workbench/bulkhaul/bulkhaul-server
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64   # 注意是 17 不是默认 21
export PATH=$JAVA_HOME/bin:$PATH
setsid nohup mvn -q -o spring-boot:run > /tmp/bulkhaul-server.log 2>&1 &
```

- 就绪判定：`curl http://127.0.0.1:8081/api/health` → `{"status":"UP","db":"connected","tables":105}`
- 日志过滤噪音：grep -vE "Preparing:|Parameters:|Columns:|Row:|Total:|Closing|Creating|JDBC Connection|SqlSession"
- scheduler.auto-enabled=false（确定性运行）：前端/E2E 需要进度推进时手动 POST /api/scheduler/tick

## 3. 全量验证跑法（当前基线）

| 套件 | 命令 | 基线 | 依赖 |
|---|---|---|---|
| flow 层（环节 1–35） | `npm test`（Windows 侧） | **556 通过 0 失败** | 无（纯 mock 层） |
| UI E2E（19 组场景） | `npm run build && node scripts/verify-ui.mjs` | **82 通过 0 失败** | dist 已构建 + 后端 8081 + Chrome/Edge |
| 操作列自适应 | `node scripts/verify-actioncol.mjs` | 5 断言 | 同 E2E |

E2E 幂等性保障（第五轮落地）：verify-ui.mjs 开头 + 关键场景前 resetDemo()（POST /admin/reset-demo），
seed_* 快照表兜底 → **跑多少次都稳定，可连续复跑验证**。

## 4. 种子污染治理（核心教训）

**问题链**：业务写操作 commitAll 回写 biz_* → 重启后 DataStore 从污染态加载并 captureSeed
→ reset-demo 也回不到正确种子 → 依赖种子态的断言（如银行自动核销）永远失败。

**解法（已固化）**：
1. `V4__seed_snapshot.sql`：biz_* 种子态固化到只读 seed_* 表（34 张，payload JSON）；
2. `DataStore.tryLoadSeedFromSnapshot()`：种子基线优先从 seed_* 加载，seed_* 缺失回退内存捕获（旧库兼容）；
3. 恢复种子态：重放 `scripts/V3__biz_tables.sql`（DROP+INSERT 全量重建 biz_*）。

**纪律**：seed_* 只读，业务永不写；改种子必须同步 V3（biz_*）+ V4（seed_*）两份 SQL 并重放。

## 5. 时间敏感断言教训（E2E 稳定性）

1. **不写死种子实体**：断言"某客户有已结算/逾期账单"这类，用动态定位
   （筛"有预付款 + 有已结算/逾期未付清账单"的客户），种子漂移后不脆。
2. **eta 是 dump 时刻的固定过去时间**：种子在途车次到 E2E 运行时必然延误，
   前端 isDelayed 全命中 → 监控页"在途"过滤不到。需要新在途车次时**动态创建**
   （confirmLoad + depart，eta=未来时间），不依赖种子在途态。
3. **headless 下浏览器 timer 脆弱**：进度推进由 node 侧手动驱动 /api/scheduler/tick
   （确定性），观察用后端快照 float 比较（UI 取整显示会 flaky）。
4. **列表项定位**：铁路车次 vehicleId=null 时 plate 映射不可靠 → 加
   data-dispatch-id 属性做 E2E 锚点（track/index.vue）。
5. **401/403 page-error 是预期噪音**：RBAC 拦截场景的断言通过即正常，
   日志里的 Unauthorized/Forbidden 不影响结果判定。

## 6. 流程纪律（fix-plan 协议）

- fix-plan.md（同目录）是唯一事实来源：每项 先实现 → 补断言 → npm test 全绿 → 才标 done-verified；
- 上下文压缩后：重读 fix-plan → 找第一个非 done-verified 项 → 先跑测试判断现状再动手；
- 文档标 done 但测试红 = 不可信（第四轮教训：M3 崩溃掩盖 8 个失败）；
- 遗留项必须写进 fix-plan 对应轮次"遗留"小节，不许口头带过。

## 7. 已知取舍（不实现，见 fix-plan F8）

异常损失单一金额字段 / 客户门户不能自助预付 / 银行自动核销仅精确匹配。

## 8. reset-demo 与实时模拟的交互（2026-08-31 阶段0 复验发现）

**现象**：调用 POST /api/admin/reset-demo 后等待数秒再比对 biz_* vs seed_*，
biz_dispatches 会出现 ~10 行 payload 差异（progress 93→93.34 漂移、异常单幂等重报，
op_log 中 operator=未登录 的"上报异常"）。

**根因**：重置本身是原子正确的（重置后**立即**比对 = 0 差异，已实测）；差异来自
运行栈的前端每 3s 轮询 POST /api/scheduler/tick 在重置后继续推进实时模拟
（scheduler.auto-enabled=false 时由前端驱动 tick）。重置=恢复演示基线，演示随即继续跑，
属设计内行为，非缺陷。

**验证纪律**：验证"重置是否持久化回种子态"时，必须在**同一脚本内重置后立即比对**
（无 sleep、无中间 tick），否则会把模拟推进误判为重置失败。

    curl -s -o /dev/null -X POST http://127.0.0.1:8081/api/admin/reset-demo -H "Authorization: Bearer $TA"
    mysql -ublms -pblms123456 -h127.0.0.1 blms -N -e "
      SELECT (SELECT count(*) FROM biz_dispatches a JOIN seed_dispatches b ON a.id=b.id WHERE a.payload<>b.payload);"
    # 期望 0

**运行栈 RBAC 实测基线（2026-08-31）**：非管理员 403（forbidden，服务层拦截）/
管理员 200（note 含"已回写 DB（持久化）…leader 租约已清空"）/ 未登录 401（unauthenticated）。

## 9. Phase 4 阶段 3 E2E 两处排序/时序敏感性（2026-08-31 阶段3 复验发现）

### 9.1 消息中心"免打扰"标记（环节6）：列表排序把系统消息挤出第 1 页

**现象**：verify-ui 环节6 保存免打扰（屏蔽"系统"类型）后，断言"列表出现免打扰标记"
10s 超时。后端 dnd 已正确落库（快照 dnd.admin.enabled=true/mutedTypes=[system]），
3s 定时任务快照刷新也正常，标记逻辑（isMuted：类型屏蔽或时段命中）无误——但标记不出现。

**根因**：消息视图读**未排序**的本地 db.messages（visibleMessages 原样返回），分页 10/页。
快照按"最新在前"排序，定时任务每 tick 生成的异常消息（时间戳最新）不断把唯一的系统消息
（种子 MSG-0010）往后挤——运行越久，系统消息越靠后（实测已挤到第 25 位/第 3 页）。
断言只查第 1 页行 → 永远看不到标记。**与代码改动无关**（消息视图不走 useCollection/
blms:refreshed，读本地 db），是既有排序敏感性随运行时长暴露。

**修复（测试侧，最小化）**：断言前先按"系统"类型筛选（.filter-bar .el-select 选"系统"），
列表只剩系统消息 → 必在第 1 页，标记断言与排序解耦。另注意标记依赖**本地** db.dnd：
保存后 200ms 防抖快照刷新可能早于后端 PUT 落库（commitAll ~2.3s）而回写种子态，
需等下一轮 3s 定时任务刷新把已落库 dnd 同步回本地标记才出现（10s 窗口足够覆盖）。

### 9.2 P2 在途监控（场景13）：验证后端必须 SCHEDULER_AUTO_ENABLED=false

**现象**：重建后端后 P2 两断言失败（监控页无可观察在途车次 / 进度不自动推进）。
场景13 动态创建在途车次（depart 生成未来 eta）后由 node 侧手动驱动 /api/scheduler/tick
观察进度推进（UI 只读）。

**根因**：验证后端误以 SCHEDULER_AUTO_ENABLED=true 运行（compose 默认值，生产口径）。
后端 3s 自动 tick 与测试手动 tick 叠加，围栏 delay 分支在观察窗口内把新在途车次转成
exception → 无可观察目标。verify-ui.mjs 场景13 注释明确：验证/演示环境确定性运行
（auto-enabled=false，node 侧手动驱动 tick 等价前端 timer）。

**纪律**：重建验证后端必须显式 -e SCHEDULER_AUTO_ENABLED=false（recreate_backend.sh /
start_backend.sh 已固化）；生产部署才用 true（C4 leader 单实例）。区分"验证栈"与"生产栈"
的定时任务开关，避免确定性 E2E 被后台 tick 干扰。

