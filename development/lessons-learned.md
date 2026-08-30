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
