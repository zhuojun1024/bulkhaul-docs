# 监控与告警说明

> BulkHaul 运维文档 · V1.0（2026-09-02）· 演示定位（当前已具备 + 生产建议清单）

## 1  当前已具备

| 能力 | 实现 | 位置 |
|---|---|---|
| 健康检查 | GET /api/health → {status, db:connected, tables:105} | actuator health（C3） |
| 指标 | actuator metrics（JVM/HTTP/DB 连接池） | /api/actuator/metrics |
| 结构化日志 | JSON 日志含 traceId（跨请求追踪） | 容器 stdout（docker logs） |
| 容器自愈 | restart: unless-stopped（异常退出自动重启） | compose |
| 启动依赖 | healthcheck + depends_on（mysql/redis → backend → frontend） | compose |

## 2  建议告警项（接 Prometheus/Grafana 或云监控）

| 级别 | 告警 | 条件 | 处理 |
|---|---|---|---|
| P0 | 后端不可用 | /api/health 非 UP 或 5xx 率 > 5%（1 分钟） | 查 docker logs；按 runbook 3.3 重建 |
| P0 | 数据库断连 | health db:disconnected | 查 mysql 容器；volume 空间 |
| P1 | 容器重启 | restart 计数增加 | 查 OOM/异常栈 |
| P1 | 磁盘水位 | 数据盘 > 80% | 清理/扩容（dump 保留策略） |
| P2 | 登录失败激增 | 单 IP 失败 > 10 次/分钟 | 限流已生效（429）；观察是否攻击 |
| P2 | 409 冲突激增 | conflict 率 > 1% | 并发写热点；检查前端防抖 |
| P2 | 快照体积 | /snapshot 响应 > 5MB | 数据增长；评估增量同步 |

## 3  日志采集（生产建议）

- docker logs → 日志平台（Loki/ELK）；JSON 结构化已就绪（traceId 关联）；
- 保留策略：应用日志 14 天；审计日志（DB 持久化）长期保留；
- 敏感信息：日志不含密码/token 明文（JWT 仅 header 传递，日志脱敏）。

## 4  演示定位说明

演示/验收栈不部署监控组件（成本/复杂度不匹配）；健康检查 + 容器自愈 + 结构化日志已覆盖"坏了能发现、能定位"的最低要求。转生产时按第 2/3 节接监控平台。
