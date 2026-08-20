# SGLang-DAS 故障诊断知识库

面向海光 DCU / HCU 上的 SGLang-DAS 部署测试，覆盖 IFB、PD 分离、TP / DP / CP / EP / PP、量化模型、LightOp、DeepEP / RocSHMEM、Mooncake 和 DSpark。

## 核心原则

1. **阶段优先，signature 次之**：先判断失败发生在哪个阶段，再用错误串缩小候选集。
2. **症状不等于根因**：`EOFError`、`SIGQUIT` 等下游信息通常只说明子进程已退出。
3. **结论必须可验证**：不能因为 signature 命中就直接判定根因。
4. **静默失败同样建模**：进程消失、配置静默回退、性能下降使用 `detection` 描述。
5. **知识与 Skill 解耦**：Wiki 保存知识；Skill 只保存查询和沉淀流程。

## 快速开始

```bash
# Wiki 默认位置
export SGLANG_KB_ROOT="$HOME/sglang-wiki"

# 两个 Skill 已放到 Claude Code 用户级目录
ls "$HOME/.claude/skills/sglang-triage/SKILL.md"
ls "$HOME/.claude/skills/sglang-kb-record/SKILL.md"
```

查询入口：

- 人工浏览：[index.md](index.md)
- 机器首查：[index.tsv](index.tsv)
- 领域规律：[heuristics.md](heuristics.md)
- 取证流程：[playbooks/evidence-collection.md](playbooks/evidence-collection.md)
- 首错定位：[playbooks/find-first-error.md](playbooks/find-first-error.md)

## 用户只粘贴报错时

`sglang-triage` 不会立刻要求用户重新整理材料。它会依次：

1. 从粘贴内容中识别日志路径、启动脚本路径和重定向表达式；
2. 能访问启动脚本时，读取其中的 `tee`、`LOG_FILE`、`> ... 2>&1`、`nohup -o` 等配置并定位日志；
3. 找到唯一日志后直接读取完整文件；
4. 找不到时，一边基于粘贴片段给出 `suspected` 级初判，一边一次性请求缺失证据。

仅有粘贴片段的结论不得标记为 `confirmed`，也不得直接沉淀进 Wiki。

## 仓库结构

```text
sglang-wiki/
├── AGENT.md
├── README.md
├── index.md
├── index.tsv
├── log.md
├── heuristics.md
├── baseline/
├── symptoms/
├── failures/
├── playbooks/
├── components/
└── raw/
```

## 维护

- 解决问题后，手动触发 `sglang-kb-record`。
- 先生成草稿和 diff；人工确认后才写入。
- Git commit、push 和 PR 均需用户明确授权。
- `--lint` 默认只报告，不自动修改条目状态。
