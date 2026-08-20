---
name: sglang-kb-record
description: 仅当用户明确要求“记录、沉淀、写入、更新、lint 或 bootstrap SGLang-DAS 故障知识库”时使用。把已解决且有证据的问题转成 symptom/failure 页面，更新 index.md、index.tsv 和 log.md；若当前问题暴露出具体的诊断或沉淀流程缺陷，可在同一草稿中完善 Wiki 内分发的 Skill。人工确认后写入并自动创建本地 git commit；不自动 push 或创建 PR。
---

# SGLang KB Record

## 职责

把已解决问题转为可复用、可验证、可审查的 Wiki 知识。不是 triage 工具，不在问题尚未定位时抢先写结论。`$KB/skills/` 保存两个 Skill 的可分发规范副本；它不属于诊断知识，但当前案例若暴露出明确流程缺陷，可以在同一次审核中改进对应 Skill。

## 路径

```bash
if [ -n "$SGLANG_KB_ROOT" ] && [ -d "$SGLANG_KB_ROOT" ]; then
  KB="$SGLANG_KB_ROOT"
elif [ -d "$HOME/sglang-wiki" ]; then
  KB="$HOME/sglang-wiki"
else
  echo "SGLang KB 未安装；请 clone 仓库并设置 SGLANG_KB_ROOT"
fi
```

开始前读取：

- `$KB/AGENT.md`
- `$KB/index.md`
- `$KB/index.tsv`
- 候选 symptom/failure 页面
- `git -C "$KB" status --short --branch`

`$KB/skills/` 默认不参与根因搜索或知识匹配。只有评估具体 Skill 改进、校验分发副本或同步安装副本时才读取。

发现已有无关改动时，保留并明确列出；不得覆盖、回滚或把它们混入后续 commit。

## 模式

- 默认：从当前会话沉淀一个已解决问题。
- `bootstrap`：调查源码和真实日志，生成首批种子知识。
- `lint`：校验 schema、索引、引用和 verify；默认只报告。

---

## 默认模式

### 1. Evidence Gate

至少需要：

- 完整日志或可复现的最小用例；
- 最早有效错误/静默 detection；
- 当前环境或版本边界；
- 定位过程；
- 修复/规避及验证结果。

证据等级：

- A：可创建 `confirmed` 候选；
- B：只能创建 `suspected` 候选；
- C（只有聊天框粘贴片段）：**拒绝写入**，提示先补完整日志或复现实验。

不要把“修复后没再看到”单独当 confirmed；必须说明哪项验证区分了候选根因。

### 2. 判断新建还是合并

先查：

1. `index.tsv` 的归一化 signature；
2. 同 layer/components 的 failure；
3. symptom 的 `possible-causes`；
4. 已有页面的根因和复发判据。

规则：

- 同一机制、不同表层错误 → 合并到同一 failure，补 signatures / masquerades-as；
- 同一症状、不同机制 → 新建/更新 symptom 的 possible-causes，不合并 failure；
- 同一修复但根因不同 → 保持独立 failure；
- 不确定是否同根因 → 新条目标 `suspected`，并在定位过程写待排项。

### 3. 归一化 signature

去掉：

- 时间戳、PID、rank/local rank；
- 内存地址；
- 绝对路径前缀（保留稳定尾部和文件名）；
- 易变行号（若版本间变化大，使用 `[0-9]+`）；
- 显存总量、free、allocation 数值；
- 请求 id、IP、token、用户数据。

保留：

- 异常类；
- 稳定函数/API；
- 组件/文件名；
- 稳定错误短语。

每条 signature 必须有 `signature-sources`：

- 首选 `repo-relative-path:line`；
- 外部库源码不在 checkout 时写 `external:<component>:<path:line>`；
- 不能找到真实来源时，不得编造，改用 `detection` 并说明未验证。

### 4. 生成页面草稿

严格遵守 `$KB/AGENT.md` schema。

Failure 必填逻辑：

- `signatures` 与 `detection` 至少一个非空；
- `layer` 只取允许枚举；
- `masquerades-as` 只引用已有 symptom id；
- `repro-condition` 区分 `observed:` 与已稳定复现；
- `applies-to` 写清版本边界，未知就写 unknown，不猜；
- `verify` 定义 `PASS / FAIL / NOT-APPLICABLE / ENV-MISMATCH`；
- `sources` 指向脱敏证据。

正文必须有：

```markdown
## 现象
## 根因
## 定位过程
## 修复 / 规避
## 复发判据
```

`定位过程` 必须写哪一步排除了什么，这是最高价值内容。

### 5. 同步变更集

每次沉淀至少评估：

- `symptoms/<id>.md`
- `failures/<id>.md`
- `index.md`
- `index.tsv`
- `log.md`
- `raw/<evidence>.md`（仅脱敏快照）
- 必要时 `components/` 或 `playbooks/`
- 当前案例确实暴露流程缺陷时，`skills/sglang-triage/SKILL.md` 或 `skills/sglang-kb-record/SKILL.md`

`index.tsv` 格式固定：

```text
<normalized-regex>\t<target-id>\t<one-line-conclusion>
```

不要添加 grep 不支持的 PCRE lookahead；保持可由普通 ERE/regex 处理。

### 5.1 同时评估 Skill 是否需要改进

每次沉淀都做一次轻量复盘：这次定位是否暴露了现有 Skill 的具体不足？例如：

- 没有覆盖一种常见日志/启动脚本定位方式；
- 阶段路由或最小实验顺序不合理；
- 某条命令在真实环境无效、不安全或容易误判；
- 输出契约、signature 归一化、verify 或提交步骤缺失；
- 同类问题已多次需要人工补充同一流程。

若有明确依据，将对应修改加入本次草稿：

- 诊断流程 → `$KB/skills/sglang-triage/SKILL.md`
- 沉淀/维护流程 → `$KB/skills/sglang-kb-record/SKILL.md`

规则：

- Wiki 内 `skills/` 是可分发规范副本，也是 Git 中的 Skill 内容来源；
- Skill 只保存流程，不把当前 case 的 signature、根因和版本事实复制进 Skill；这些仍写入 symptom/failure/playbook；
- 只修改与当前案例证据直接相关的最小范围，不做顺手重写或纯风格调整；
- 在草稿中单独说明“暴露的问题 → Skill 修改 → 为什么能防止复发”；
- Skill 修改与知识修改一起展示、一起确认、一起进入同一 commit；
- 若没有具体改进点，明确写“本次无需修改 Skill”，不要为了更新而更新。

### 6. 先展示，不写入

在对话中展示：

1. 新建/合并判断；
2. status 及理由；
3. 涉及文件；
4. 完整草稿或 unified diff；
5. 尚未验证项和敏感信息处理；
6. Skill 复盘结果：无需修改，或具体 Skill diff 及其证据。

得到用户明确确认后才调用写文件工具。确认前不创建临时 Wiki 文件，不修改 index/log。

**该确认同时授权本 Skill：写入已展示的变更、完成校验并创建本地 Git commit。** 不需要再单独询问是否 commit；但确认不授权 push 或创建 PR。

### 7. 写入、验证与提交

#### 7.1 提交安全检查

- 若存在**本次操作之前就已 staged 的变更**，停止自动提交并告知用户，避免把其他人的暂存内容混入 commit。
- 未 staged 的无关改动可以保留，但不得修改或加入本次 commit。
- 若当前位于 `main` / `master` 等默认分支，在写入前自动创建独立分支：`kb/<YYYYMMDD>-<failure-id-or-topic>`；若已在非默认分支，则继续使用当前分支。
- 分支名冲突时添加简短递增后缀，不覆盖已有分支。

#### 7.2 写入后验证

- 检查 frontmatter 和必要字段；
- 检查所有相对链接/ID 存在；
- 检查 `index.md` / `index.tsv` 与页面一致；
- 检查 raw 未泄露 token/IP/业务正文；
- 运行 `git -C "$KB" diff --check`；
- 展示本次准备提交的文件清单和 diff stat。

任一校验失败时不 commit，保留修改并如实报告。

#### 7.3 只提交本次文件

使用显式路径暂存，不使用 `git add -A`、`git add .`：

```bash
git -C "$KB" add -- <本次新建或修改的精确路径列表>
git -C "$KB" diff --cached --check
git -C "$KB" diff --cached --name-status
git -C "$KB" diff --cached --stat
```

确认 cached diff 只包含已展示并获批的知识变更后，直接创建本地 commit：

```bash
git -C "$KB" commit \
  -m "kb: record <failure-id>" \
  -m "Co-Authored-By: Claude <noreply@anthropic.com>"
```

合并进已有条目时使用 `kb: update <failure-id>`；一次记录涉及多个紧密相关条目时使用简洁主题，不把无关问题合在一个 commit。仅修改 Skill 而没有知识条目时使用 `skill: improve <skill-name>`。

#### 7.4 同步当前用户安装副本

仓库 commit 成功后，若本次修改了 `$KB/skills/<name>/SKILL.md`：

1. 将该仓库文件作为规范版本；
2. 比较 `$HOME/.claude/skills/<name>/SKILL.md`；
3. 安装副本不存在时创建；与修改前规范版本一致时更新为新版本；
4. 安装副本存在额外本地定制时不覆盖，报告差异和手动同步方式。

安装副本的同步不进入 Wiki commit。正在运行的 Skill 可能要到下一次调用或新会话才按新内容生效。

commit 成功后报告 branch、commit hash、文件列表和 Skill 同步结果。**到此停止：不 push，不创建 PR。** 用户会自行推送并在 GitHub 创建 PR、完成审核。

任何会写文件的模式（默认模式、确认后的 bootstrap、确认应用 lint 修复）都遵循本提交流程；纯 lint 报告不创建 commit。

---

## bootstrap 模式

首次或阶段性扩充种子库时执行。**调查报告先于批量落盘。**

### 四层调查

1. 算子：注册方式、扩展产物、加载路径、shape/dtype/layout 契约；
2. 错误：扫描 `raise / assert / TORCH_CHECK / HIP_CHECK / CUDA_CHECK / exit / abort`，保存真实字符串和 `文件:行号`；
3. 量化：weight loader、scale、per-tensor/per-channel/group-wise、TP/EP 切分和 repack；
4. 版本：sglang / sgl-kernel / torch / DTK / lightop / git hash 的采集方式。

### 报告必须说明

- 调查仓库、branch、commit、dirty 状态；
- 当前宿主与实际测试容器是否 ENV-MISMATCH；
- 候选 symptom/failure；
- 哪些第三方源码不可读；
- 哪些结论仅来自日志；
- 拟写文件列表。

用户确认报告后再批量写入；不要求第一期铺满所有 components/configs。

---

## lint 模式

默认只报告，禁止自动改 status。

### 静态检查

- YAML/frontmatter 是否可解析；
- id 与文件名是否一致；
- layer/status/diagnostic-value 枚举；
- `signatures` 与 `detection` 是否至少一个非空；
- `signature-sources` 是否完整；
- symptom/failure 引用是否存在；
- `index.tsv` 目标是否存在；
- index 重复/冲突；
- Markdown 相对链接是否存在；
- `suspected` 是否长期未复核；
- raw 是否疑似包含密钥、token、内网地址或业务请求正文；
- `skills/*/SKILL.md` 的 frontmatter、目录名和 description 是否有效；
- Wiki 规范 Skill 与 `$HOME/.claude/skills/` 安装副本是否漂移（只报告，不覆盖本地定制）。

### verify 四态

- `PASS`：断言在适用环境成立；
- `FAIL`：断言在适用环境不成立，需要人工复核；
- `NOT-APPLICABLE`：当前版本/功能不适用；
- `ENV-MISMATCH`：依赖、GPU、DTK、节点角色等环境不同。

`FAIL` 不等于知识失效；先区分环境问题和实际代码变化。输出建议，不修改 `status`。

### lint 输出

```markdown
## Summary
- total / pass / fail / not-applicable / env-mismatch

## Schema / index findings
- file:line — finding — suggested action

## Verify findings
- id — state — evidence

## Suggested changes
- 仅建议，不自动应用
```
