# bulkhaul-docs

大宗物流综合管理平台（bulkhaul）项目级文档仓库。

> 代码仓库：`bulkhaul-manage-web`（前端）、`bulkhaul-server`（后端）。
> 本仓库只放**跨仓库 / 项目级**文档；纯前端文档（如 design-system）留在前端仓库 `docs/`。

## 目录结构

| 目录 | 用途 | 状态 |
|---|---|---|
| `development/` | 开发文档：架构规划、接口联调、整改进度、业务审计、验证经验、生产化落地方案 | 已有 |
| `requirements/` | 需求文档：业务需求、功能规格、变更单 | 预留 |
| `training/` | 培训文档：操作手册、演示脚本、新人指引 | 预留 |
| `acceptance/` | 验收文档：验收标准、验收报告、问题清单 | 预留 |

## development/ 索引

| 文档 | 说明 |
|---|---|
| `backend-plan.md` | 后端架构规划（前端 → 完整后端服务的唯一规划来源） |
| `api-integration-plan.md` | 前端切真实 API 联调（阶段 6 收尾，W 映射 97 端点） |
| `fix-plan.md` | 修复进度计划（P0–P3 全环节整改，含压缩后重入协议） |
| `flow-audit.md` | 业务闭环审计结论（流程合理性/完整性评估） |
| `lessons-learned.md` | 验证与运维经验（环境拓扑/后端启动/全量验证跑法/断言纪律） |
| `production-readiness-plan.md` | 生产化落地方案（安全/架构/运维/开发 16 项缺口，P0–P1 分 Phase） |
