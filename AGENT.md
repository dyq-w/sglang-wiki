# SGLang-DAS KB Agent Contract

## 红线

> **结论必须来自可复现证据，不允许因为 signature 命中而直接认定根因。**

- `confirmed`：已具备现象、首错证据、根因解释、排他过程和验证实验。
- `suspected`：有证据支持，但未完成排他验证；仅有粘贴片段时必须使用此状态。
- `fixed-in <commit/version>`：已知修复边界，仍保留历史条目但不作为新版本的首选答案。
- 阶段优先，signature 次之。
- 先找最早异常，禁止从日志尾部倒推根因。
- `diagnostic-value: none` 的症状不能直接成为诊断结论。
- 仅有粘贴片段时不得写入知识库；先补齐完整日志或可复现实验。

## 路径

按顺序解析 Wiki 根目录：

1. `$SGLANG_KB_ROOT`
2. `$HOME/sglang-wiki`
3. 均不存在时，提示用户安装知识库，不得静默回退到模型记忆。

## Failure schema

```yaml
---
id: required-kebab-case
kind: failure
layer: launch | weights-quant | kernel | distributed | pd | memory | accuracy | perf
components: []
signatures: []
signature-sources: []
detection: |
masquerades-as: []
repro-condition: []
applies-to: {}
status: confirmed | suspected | fixed-in <commit/version>
first-seen: YYYY-MM-DD
fixed-in:
verify: |
sources: []
---
## 现象
## 根因
## 定位过程
## 修复 / 规避
## 复发判据
```

约束：

- `signatures` 与 `detection` 至少一个非空。
- signature 必须归一化：去地址、PID、rank、时间戳、绝对路径前缀和易变显存数值。
- `signature-sources` 优先指向真实代码的 `repo-relative-path:line`。
- 外部运行库源码不在 checkout 时，使用 `external:<component>:<path:line>`，同时在 `sources` 中保存日志证据；不得伪造本地路径。
- `masquerades-as` 只引用已有 symptom id。
- `verify` 必须说明预期输出，并支持 `PASS / FAIL / NOT-APPLICABLE / ENV-MISMATCH` 四态。
- `FAIL` 不自动修改 `status`；先判断是否环境不一致。

## Symptom schema

```yaml
---
id: required-kebab-case
kind: symptom
signatures: []
signature-sources: []
diagnostic-value: none | weak | strong
means:
action:
possible-causes: []
---
```

## 内容职责

- `symptoms/`：用户看到的现象；不冒充根因。
- `failures/`：真实失败机制；包含验证和版本边界。
- `playbooks/`：怎么查；按失败阶段组织。
- `components/`：代码基线摘要；只写入口、契约和已验证行为，不写完整代码文档。
- `raw/`：经脱敏的证据快照或原始日志说明；禁止无审查加入密钥、令牌、内网拓扑和个人信息。
- `index.md`：人读导航。
- `index.tsv`：机器首查，一行一个归一化 regex。
- `log.md`：append-only 变更时间线。

## 写回流程

1. 读取 `index.md`、`index.tsv` 和候选已有页面。
2. 同一根因的不同表现优先合并，不无脑新建。
3. 生成草稿，不直接覆盖。
4. 展示完整 diff。
5. 用户确认后写入文件并同步两份索引和 `log.md`。
6. 写入和校验成功后，自动在独立分支创建仅包含本次知识变更的本地 commit。
7. 不自动 push 或创建 PR；由用户在 GitHub 完成 PR 和审核。

## Lint

默认只报告：

- frontmatter/schema 错误；
- `signatures` / `detection` 同时为空；
- 不存在的引用；
- index 与页面不一致；
- 同一 signature 指向多个 `confirmed` 根因；
- `suspected` 长期未复核；
- `verify` 的四态结果；
- 可能包含敏感信息的 raw 文件。

禁止 lint 自动降级或升级条目状态。
