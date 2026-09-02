# Changelog（bulkhaul-docs）

> 项目级文档仓库。

## [1.0.0] - 2026-09-02（当前，f4e8f1e）

### 交付文档体系
- **验收包**（477405f）：验收测试计划（BLMS-ACC-001）+ 验收测试用例（BLMS-ACC-002，111 自动化 + 12 人工）+ 验收报告（BLMS-ACC-003，110/110 + 12/12 通过，结论合格）
- **交付文档第一批**（f4e8f1e）：
  - requirements/需求规格说明书.docx（BLMS-REQ-001，54 功能需求 + 6 非功能 + 追踪矩阵）
  - training/用户手册.docx（BLMS-TRN-001，9 章按角色 + 30 张实测截图）
  - development/api-reference.md（97 写端点 + 读端点 + 通用约定）
  - development/数据字典.xlsx（34 集合字段，快照实测）
  - deployment/runbook.md（运维手册：日常操作/故障排查/验证跑法/备份/变更纪律）

## [0.9.0] - 2026-08-31（Phase 4 引擎移除阶段）
- production-readiness-plan：F1–F5 全部完成（Phase 4 终态 done-verified）
- E2E 环境阻塞解决（WSL 内运行 + SCHEDULER_AUTO_ENABLED=false 纪律）
- 引擎移除批次 A–F 全程进度记录（116 写操作切后端权威 → 引擎删除）

## [0.8.0] - 2026-08-30（生产化方案）
- production-readiness-plan：16 项缺口（安全/架构/运维/开发，P0–P1 分 Phase）
- C1 部署（Docker 一键起全栈）done-verified / C2 CI done-verified
- development/ 索引：backend-plan / api-integration-plan / fix-plan / flow-audit / lessons-learned
