# 运维手册（Runbook）

> BulkHaul 大宗物流综合管理平台 · 运维操作手册
> 版本 V1.0（2026-09-02）· 被测版本：前端 944c115 / 后端 ad0b06e
> 配套：deployment/README.md（部署）、development/lessons-learned.md（经验沉淀）

## 1  系统组成与端口

| 组件 | 技术 | 端口 | 说明 |
|---|---|---|---|
| 后端 bulkhaul-server | Spring Boot 3.3.5 / Java 17 | 8081 | 唯一权威态：业务状态机、RBAC、乐观锁、审计、定时任务 |
| 数据库 MySQL 8 | 容器 mysql | 3306 | blms 库，105 张表 = biz_*(34) + seed_*(34) + sys_* 等 |
| 缓存 Redis | 容器 redis | 6379 | 验证码/登录锁定/leader 租约 |
| 前端 bulkhaul-manage-web | Vue 3 + Nginx | 8080 | 生产构建 dist，/api 反代后端 |

**验收/验证栈**（开发环境，Windows + WSL 分工）：

| 端口 | 用途 |
|---|---|
| 8081 | 后端（WSL 内 Docker 容器 blms-backend，SCHEDULER_AUTO_ENABLED=false） |
| 8086 | verify-ui.mjs 静态服务（dist + /api 反代 8081） |
| 8087 | vue devServer 本地联调 |

> 纪律：**验收/验证栈 SCHEDULER_AUTO_ENABLED 必须为 false**（确定性：在途进度由 node 侧手动驱动 /api/scheduler/tick）；**生产栈为 true**（C4 leader 单实例执行）。两栈混用会导致场景 13（在途监控）断言失败。

## 2  部署（生产栈）

```bash
cd <workspace-root>          # 三仓库公共父目录
cp .env.example .env         # 覆盖生产密钥（见下）
docker compose up -d --build
# 前端 http://localhost:8080  后端健康 http://localhost:8081/api/health
```

启动顺序：mysql/redis(healthy) → backend(healthy) → frontend。Flyway V1–V4 首启自动建库建表灌种子，约 40–60s。

**生产密钥必改**：JWT_SECRET（≥32 字节随机，openssl rand -base64 48）、DB_PASSWORD、MYSQL_ROOT_PASSWORD；加 SPRING_PROFILES_ACTIVE=prod（敏感项无 dev 默认，缺 env 即启动失败）；SCHEDULER_AUTO_ENABLED=true；注释 mysql/redis 的 ports 映射（仅内网访问）。

## 3  日常操作

### 3.1  健康检查

```bash
curl -s http://127.0.0.1:8081/api/health
# 期望 {"status":"UP","db":"connected","tables":105}
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/   # 期望 200
```

### 3.2  重置演示数据（回种子态）

页面入口：管理后台 → 重置演示数据。或 API：

```bash
# 先登录取 token（admin/123456 + 图形验证码）
curl -s -X POST http://127.0.0.1:8081/api/admin/reset-demo -H "Authorization: Bearer $TOKEN"
```

- 重置是原子操作（biz_* 全量重建自 seed_* 只读快照表）；
- 重置后前端每 3s 的 scheduler tick 会继续推进演示（设计内行为）；验证"是否回到种子态"须**重置后立即比对**（无 sleep/无中间 tick）；
- 非管理员调用 → 403 forbidden；未登录 → 401。

### 3.3  重建后端容器（验证栈，WSL 内）

```bash
wsl -d Ubuntu-24.04 -- bash -lc '
  docker stop blms-backend && docker rm blms-backend
  bash /mnt/d/Documents/workbench/bulkhaul/bulkhaul-server/start_backend.sh'
# start_backend.sh 内置健康等待（40×3s）；成功输出 HEALTH UP
```

- 必须显式 -e SCHEDULER_AUTO_ENABLED=false（脚本已固化）；
- 后端日志：docker logs --tail 100 blms-backend；
- 就绪判定同 3.1。

### 3.4  前端重新构建

```bash
cd bulkhaul-manage-web
npm run build        # 产出 dist/（生产栈由 Nginx 容器构建；验证栈静态服务读 dist）
```

### 3.5  登录锁定处理

- 连续 5 次登录失败 → 账号锁定 5 分钟（Redis 计数，刷新后仍生效）；
- 处理：等 5 分钟自动解除；或 WSL 内清 Redis 键（redis-cli 删除 blms:lock:* 前缀键）；
- 管理员可后台重置密码（重置后需重新登录）。

### 3.6  定时任务驱动（验收栈）

```bash
curl -s -X POST http://127.0.0.1:8081/api/scheduler/tick -H "Authorization: Bearer $TOKEN"
# 推进在途进度/ETA/围栏判定/消息生成；前端 3s 自动轮询，node 侧手动驱动用于确定性测试
```

## 4  故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 前端"无法连接后端服务" | 后端 8081 未起/挂了 | docker ps 查 blms-backend；docker logs 看启动错误；按 3.3 重建 |
| /api/health 返回 db:disconnected | MySQL 容器异常 | docker compose ps 查 mysql；docker compose restart mysql |
| 登录页 401 控制台报错（3s 一次） | 未登录态前端 3s scheduler tick 无 token | **预期噪音**，无害；登录后消失 |
| E2E 场景 13 在途断言失败 | SCHEDULER_AUTO_ENABLED=true（生产配置跑验证栈） | 按 3.3 重建为 false |
| E2E 断言"种子数据"类失败 | 种子污染（业务写回写 biz_* 后被当种子） | 重放 scripts/V3__biz_tables.sql 重建 biz_*；确认 seed_* 只读未被写 |
| 重置演示后数据"又变了" | 重置后 tick 继续推进演示（设计内） | 验证须重置后立即比对；非缺陷 |
| 页面"数据已变更，请刷新"toast | 乐观锁 409（expectedVersion 不匹配） | 用户刷新页面即恢复（前端已自动拉回权威态） |
| 消息中心某类消息"消失" | 列表排序 + 分页：新消息把旧消息挤出当前页 | 用类型筛选定位；非缺陷（消息视图读本地 db 未排序） |
| 后端启动慢/超时 | 首启 Flyway 迁移 + 种子灌入 | 等待（40–60s）；docker logs 观察迁移进度 |
| 前端 502 | Nginx 反代后端不通 | 确认 backend 容器 healthy；compose 内网名称解析（backend:8081） |

## 5  验证跑法（回归基线）

| 套件 | 命令 | 基线 | 依赖 |
|---|---|---|---|
| 数据层 | npm test（verify-api 35 + verify-collection 20） | 55 通过 | 无（纯 Node） |
| 前后端契约 | npm run test:contract | 97/97 | 后端源码目录（解析 Controller 路由） |
| UI E2E | npm run build && node scripts/verify-ui.mjs | 110 通过 0 失败 | dist + 后端 8081（SCHEDULER_AUTO_ENABLED=false）+ Chromium |
| 后端测试 | mvn test（WSL 内） | 33 通过 | JDK 17 + Maven（离线 -o） |

E2E 幂等：脚本开头 + 关键场景前 resetDemo()，seed_* 快照兜底，可连续复跑。

## 6  数据与备份（最小版，演示定位）

- 数据卷：mysql-data / redis-data（compose 命名卷）；
- 备份：mysqldump blms 库（含 biz_*/seed_*/sys_*），建议每日一次，保留 7 份；
- 恢复：mysql blms < dump.sql 后重启 backend（Flyway 版本表随库恢复，不重放迁移）；
- 种子重建（不依赖备份）：重放 V3__biz_tables.sql（biz_*）+ 确认 seed_* 只读快照完整；
- RTO/RPO（演示定位）：RPO ≤ 24h（每日 dump），RTO ≤ 1h（重建容器 + 恢复 dump）。

## 7  变更纪律

1. 改种子数据必须同步 V3（biz_*）+ V4（seed_*）两份 SQL 并重放；seed_* 只读，业务永不写；
2. 前后端契约变更：改 endpoints.js（前端）后必须跑 test:contract 确认后端路由同步（97/97）；
3. 任何代码变更的验证门槛：build + npm test 55 + contract 97 + E2E 110（前端）/ mvn 33（后端）；
4. 生产密钥（JWT_SECRET/DB_PASSWORD）不进仓库、不进日志；prod profile 缺 env 即启动失败是设计行为。
