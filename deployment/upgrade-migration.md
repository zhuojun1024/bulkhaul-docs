# 升级与迁移手册

> BulkHaul 运维文档 · V1.0（2026-09-02）

## 1  版本构成

| 组件 | 版本载体 | 说明 |
|---|---|---|
| 后端 | git tag/commit + Flyway 迁移版本（V1–V4） | 数据库结构由 Flyway 管理（user 表 flyway_schema_history） |
| 前端 | git tag/commit | 纯静态构建，无状态，随部署替换 |
| 数据库 | blms 库（105 张表） | biz_*(34) + seed_*(34) + sys_* + Flyway 元数据 |

## 2  数据库迁移机制（Flyway）

- 迁移脚本：bulkhaul-server/src/main/resources/db/migration/V{n}__{name}.sql（只增不改）
- 首启自动执行：建库建表 + 灌种子（V1 基础结构 / V2 鉴权表 / V3 业务表+种子 / V4 种子只读快照）
- 升级时：新 V{n} 脚本自动应用；已应用版本不重放（checksum 校验）
- **纪律**：已发布的 V{n} 永不修改（改 = 新 V{n+1}）；改种子必须同步 V3（biz_*）+ V4（seed_*）

## 3  升级步骤（滚动升级，停机窗口 < 10 分钟）

```bash
# 0. 前置：备份（见 backup-restore.md）
mysqldump ... blms | gzip > /backup/blms-pre-upgrade-$(date +%Y%m%d).sql.gz

# 1. 拉取新版本（三仓库）
cd bulkhaul-server && git pull && cd ..
cd bulkhaul-manage-web && git pull && cd ..

# 2. 重建镜像
docker compose build backend frontend

# 3. 停旧起新（Flyway 自动应用新迁移）
docker compose up -d --no-deps backend
# 等待健康（Flyway 迁移 + 启动，约 40–60s）
for i in $(seq 1 40); do
  curl -s http://127.0.0.1:8081/api/health | grep -q '"status":"UP"' && break
  sleep 3
done

# 4. 前端（无状态，直接替换）
docker compose up -d --no-deps frontend

# 5. 验证（升级后检查清单）
curl -s http://127.0.0.1:8081/api/health          # tables 数符合预期
# 浏览器：admin 登录 → 工作台 → 抽 3 个核心页面（合同/调度/结算）
# 跑回归门：npm test 55 + test:contract 97 + E2E 110（见 runbook 验证跑法）
```

## 4  回滚

| 场景 | 操作 |
|---|---|
| 应用故障（无 DB 结构变更） | docker compose 回退旧镜像 tag；数据不动 |
| 应用故障（含 DB 结构变更） | 恢复升级前 dump（backup-restore.md 3.1）+ 回退镜像；Flyway 版本表随 dump 恢复 |
| 种子数据异常 | 重放 V3__biz_tables.sql 重建 biz_*（seed_* 完整时） |

> 注意：Flyway 不支持"回退迁移"（无 down migration）；结构变更回滚 = 恢复 dump。这是演示定位的取舍（生产可引入回滚脚本或双版本兼容期）。

## 5  升级后检查清单

- [ ] /api/health：status UP + tables 数正确
- [ ] 登录（admin + 验证码）→ 工作台渲染
- [ ] 核心链路抽测：合同列表 / 调度详情 / 结算列表
- [ ] 回归门：npm test 55 / test:contract 97 / E2E 110（或 mvn 33 如涉及后端）
- [ ] 操作日志无异常（403/409 突增排查权限/并发）
