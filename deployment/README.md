# 部署（C1：Docker 一键起全栈）

> 本目录是**参考副本** + 部署指南。实际运行的 `docker-compose.yml` 放在
> **三个仓库的公共父目录**（workspace root，`D:\Documents\workbench\bulkhaul\`），
> 因为 compose 用 `./bulkhaul-server` / `./bulkhaul-manage-web` 两个 build context。
> 本目录副本用于版本化留档与部署核对。

## 工件清单

| 工件 | 位置 | 作用 |
|---|---|---|
| 后端 Dockerfile | `bulkhaul-server/Dockerfile` | Maven 多阶段构建 → JRE 运行（非 root，8081） |
| 后端 .dockerignore | `bulkhaul-server/.dockerignore` | 排除 target/.env/IDE/VCS |
| 前端 Dockerfile | `bulkhaul-manage-web/Dockerfile` | Node 构建 → Nginx 托管 + /api 反代 |
| 前端 nginx 模板 | `bulkhaul-manage-web/deploy/nginx.conf.template` | envsubst 替换 BACKEND_UPSTREAM；/api 反代；SPA 兜底；gzip |
| 前端 .dockerignore | `bulkhaul-manage-web/.dockerignore` | 排除 node_modules/dist |
| docker-compose.yml | **workspace root**（本目录留副本） | 编排 mysql + redis + backend + frontend |
| .env.example | **workspace root** + 本目录 | 全栈环境变量模板（DB/JWT/调度/日志） |

## 一键起全栈

    cd <workspace-root>          # 三仓库公共父目录
    cp .env.example .env         # 填生产密钥（本地可用默认值）
    docker compose up -d --build
    # 前端 http://localhost:8080
    # 后端 http://localhost:8081   健康 http://localhost:8081/api/health

启动顺序（depends_on + healthcheck 保证）：
`mysql/redis(healthy) → backend(healthy) → frontend`。
后端 Flyway V1–V4 首启自动建库建表灌种子；首次启动约 40–60s。

## 生产要点

1. **密钥**：`JWT_SECRET`（≥32 字节随机，`openssl rand -base64 48`）、`DB_PASSWORD`、
   `MYSQL_ROOT_PASSWORD` 必须覆盖默认值。
2. **后端生产 profile**：加 `SPRING_PROFILES_ACTIVE=prod`（或 `-e SPRING_PROFILES_ACTIVE=prod`）
   启用 `application-prod.yml`——敏感项**无 dev 默认**，缺 env 即启动失败，杜绝误用演示密钥上线。
3. **定时任务**：`SCHEDULER_AUTO_ENABLED=true`（C4 落地后 leader 单实例执行；当前单实例直接开）。
4. **端口暴露**：生产建议注释 mysql/redis 的 `ports` 映射（仅 compose 内网访问），
   只暴露 frontend 8080（或改 80）+ 必要时 backend 8081。
5. **反代/HTTPS**：前置 Nginx/Traefik 做 TLS；本 compose 的 frontend 已含 /api 反代，
   生产可只暴露 frontend，backend 不对外（见 C5 部署拓扑）。
6. **数据持久化**：`mysql-data` / `redis-data` 命名卷；备份策略见运维手册。

## 验证（起栈后）

    curl -s http://localhost:8081/api/health        # 后端健康
    curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/   # 前端 200
    # 浏览器登录 admin/123456 → 工作台（Flyway 种子 + 行级数据权限 A1 生效）

> 注：当前环境无 Docker daemon，本工件按构建系统/官方镜像规范编写，
> 首次在有 Docker 的机器上 `docker compose up -d --build` 时做端到端验证（C1 done-verified 门槛）。
