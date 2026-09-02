# 安全说明

> BulkHaul 交付文档 · V1.0（2026-09-02）· 演示定位（转生产升级项见文末）

## 1  认证

| 项 | 实现 |
|---|---|
| 密码存储 | bcrypt 哈希（sys_user 表，不落明文）；演示账号统一 123456（交付前须重置） |
| 会话 | JWT（HS256），480 分钟有效（JWT_TTL_MINUTES 可配）；Authorization: Bearer 传递 |
| 图形验证码 | SVG 渲染，一次性，60 秒有效（Redis 存储） |
| 失败锁定 | 连续 5 次失败锁定 5 分钟（Redis 计数，持久化，刷新不失效） |
| 登出 | 清除本地 token；审计日志不记在旧用户名下 |

## 2  授权（RBAC 四层，默认拒绝）

| 层 | 实现 | 绕过后果 |
|---|---|---|
| 菜单级 | ROLE_MENUS + 前端路由守卫（强制重定向） | 体验层；直调 API 不受此层保护 |
| 按钮级 | ROLE_ACTIONS + usePerm（按钮显隐） | 体验层；同上 |
| 行级数据权限 | dataScopes 按装货侧场站区域，后端 /coll 过滤 | 后端执行，不可绕过 |
| 后端单点校验 | requireAction 每写端点按操作人角色校验 | **权威层**：前端绕过 → 403 forbidden |

权限表数据化：rolePerms 集合（角色管理页可编辑，即时生效）；内置默认值见 permission-table.js（纯数据零依赖）。

## 3  数据安全

| 项 | 实现 |
|---|---|
| 传输 | 前端 /api 经反向代理（生产建议前置 TLS，见 deployment/README.md 生产要点 5） |
| 存储 | MySQL（业务数据 biz_* JSON payload）；敏感配置走环境变量（prod profile 缺 env 即启动失败） |
| 并发一致 | 乐观锁 expectedVersion（409 conflict，防并发覆盖） |
| 审计 | 全量操作日志（操作人/时间/操作/对象），操作日志页可查 |
| PII | 司机手机号（登录账号 + 档案）：行级权限限制可见范围（仅本司机/授权角色）；演示数据为虚构号码 |

## 4  密钥管理

| 密钥 | 要求 | 现状 |
|---|---|---|
| JWT_SECRET | ≥32 字节随机（openssl rand -base64 48） | 演示默认值（交付前必须更换） |
| DB_PASSWORD | 强密码 | 演示 blms123456（交付前必须更换） |
| MYSQL_ROOT_PASSWORD | 强密码 | 同上 |

prod profile（SPRING_PROFILES_ACTIVE=prod）：敏感项无 dev 默认，缺 env 即启动失败，杜绝误用演示密钥上线。

## 5  已知边界（演示定位，转生产须补）

| 项 | 现状 | 生产要求 |
|---|---|---|
| 传输加密 | 演示环境 HTTP（127.0.0.1 反代） | 前置 Nginx/Traefik TLS，HSTS |
| 速率限制 | 无（仅登录失败锁定） | API 网关限流 + 防暴力破解 |
| 安全渗透 | 未做 | OWASP Top 10 渗透测试 |
| 密码策略 | 演示统一密码 | 复杂度/过期/历史密码策略 |
| 会话吊销 | JWT 到期自然失效（480 分钟） | 登出即失效需黑名单/短 TTL + 刷新令牌 |
| 依赖漏洞 | 未扫描 | 定期 npm audit / mvn dependency-check |
