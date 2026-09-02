# 备份与恢复策略

> BulkHaul 运维文档 · V1.0（2026-09-02）· 演示定位最小版（转生产需按 production-readiness-plan 升级）

## 1  备份对象

| 对象 | 位置 | 内容 | 重要性 |
|---|---|---|---|
| MySQL 数据卷 | compose 命名卷 mysql-data | blms 库：biz_*(34 业务) + seed_*(34 种子只读) + sys_*(鉴权) + Flyway 版本表 | 核心（唯一业务数据源） |
| Redis 数据卷 | compose 命名卷 redis-data | 验证码/登录锁定/leader 租约（可再生） | 低（丢=验证码失效一次） |
| 应用镜像 | Docker 镜像 bulkhaul-backend:latest / bulkhaul-frontend:latest | 可由仓库源码重建 | 中（重建成本 5–10 分钟） |
| 源码 | 三个 git 仓库（GitHub 远端） | 全部代码与文档 | 核心（本地损坏可重拉） |

## 2  备份策略（演示定位）

| 项 | 策略 |
|---|---|
| 频率 | 每日一次（mysqldump 全库） |
| 保留 | 7 份滚动（按日期命名） |
| 存储 | 独立磁盘/目录（不与数据卷同盘） |
| 方式 | 逻辑备份（mysqldump --single-transaction，不锁表） |

```bash
# 每日备份（cron 或手动）
mysqldump -h <mysql-host> -ublms -p'<DB_PASSWORD>' --single-transaction --routines blms \
  | gzip > /backup/blms-$(date +%Y%m%d).sql.gz
# 保留 7 份
ls -1t /backup/blms-*.sql.gz | tail -n +8 | xargs -r rm
```

## 3  恢复步骤

### 3.1  从 dump 恢复（整库）

```bash
# 1. 停后端（避免写入）
docker compose stop backend
# 2. 恢复
gunzip -c /backup/blms-YYYYMMDD.sql.gz | mysql -h <mysql-host> -ublms -p'<DB_PASSWORD>' blms
# 3. 重启（Flyway 版本表随库恢复，不重放迁移）
docker compose up -d backend
# 4. 验证
curl -s http://127.0.0.1:8081/api/health   # 期望 tables:105
# 5. 冒烟：登录 admin → 工作台 → 重置演示数据（可选）
```

### 3.2  种子重建（不依赖 dump，biz_* 污染时）

```bash
# 重放 V3 全量重建 biz_*（seed_* 只读快照必须完整）
mysql -ublms -p'<DB_PASSWORD>' blms < bulkhaul-server/scripts/V3__biz_tables.sql
# 验证：seed_* 完整（34 张）
mysql -ublms -p'<DB_PASSWORD>' blms -N -e "SELECT count(*) FROM information_schema.tables WHERE table_name LIKE 'seed_%';"
# 期望 34
```

> 纪律：seed_* 只读，业务永不写；改种子必须同步 V3（biz_*）+ V4（seed_*）两份 SQL 并重放（见 runbook 变更纪律）。

## 4  RTO / RPO（演示定位）

| 指标 | 目标 | 依据 |
|---|---|---|
| RPO（数据丢失窗口） | ≤ 24h | 每日 dump |
| RTO（恢复时间） | ≤ 1h | 重建容器（5–10 分钟）+ 恢复 dump（<10 分钟）+ 验证（<10 分钟） |

## 5  转生产升级项

- 增量备份（binlog）把 RPO 降到分钟级；
- 备份自动上传对象存储 + 定期恢复演练（每季度一次，记录 RTO 实测）；
- 多副本（MySQL 主从/云 RDS）+ 自动故障切换；
- 备份加密（at rest）与访问控制。
