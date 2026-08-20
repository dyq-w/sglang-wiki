# 知识库时间线

## [2026-08-20] bootstrap | 初始化 SGLang-DAS 故障诊断知识库

- 建立 symptom / failure 多对多模型
- 建立按阶段组织的 playbooks
- 从两份真实日志沉淀 DeepEP IPC 和 DSpark HIP OOM 证据
- 从当前 SGLang / LightOp checkout 提取首批真实 signature 与静默失败机制
- 建立 `index.md` + `index.tsv` 双索引
- 建立 `sglang-triage` 与 `sglang-kb-record` 双 Skill

## [2026-08-20] workflow | 将双 Skill 纳入 Wiki 分发并支持持续改进

- 新增 `skills/` 目录，保存两个 Skill 的 Git 规范副本和安装说明
- 规定诊断时忽略 `skills/`，避免工具流程参与知识匹配
- `sglang-kb-record` 在每次沉淀后复盘 Skill；有具体证据时可在同一 diff 和 commit 中完善流程
- commit 后安全同步用户级安装副本；不覆盖本地定制，不自动 push 或创建 PR
