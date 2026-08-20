# Skills 分发目录

本目录随 `sglang-wiki` Git 仓库一起分发：

```text
skills/
├── sglang-triage/SKILL.md
└── sglang-kb-record/SKILL.md
```

## 重要边界

- 本目录是 **Skill 的分发与版本管理内容**，不是故障知识。
- `sglang-triage` 做问题分析、signature 匹配或全文搜索时必须忽略 `skills/`。
- 症状、根因、版本事实和具体案例只能写入 `symptoms/`、`failures/`、`playbooks/` 等知识目录，不复制进 Skill。
- 只有用户明确询问 Skill，或 `sglang-kb-record` 依据当前案例复盘诊断流程时，才读取本目录。

## 安装

仓库中的副本是规范版本。推荐使用符号链接，让后续 `git pull` 自动更新本地 Skill：

```bash
export SGLANG_KB_ROOT="${SGLANG_KB_ROOT:-$HOME/sglang-wiki}"
mkdir -p "$HOME/.claude/skills"

for name in sglang-triage sglang-kb-record; do
  target="$HOME/.claude/skills/$name"
  source="$SGLANG_KB_ROOT/skills/$name"
  if [ -e "$target" ] || [ -L "$target" ]; then
    echo "skip existing: $target"
  else
    ln -s "$source" "$target"
    echo "installed: $target -> $source"
  fi
done
```

已有安装副本时先比较，不要直接覆盖本地定制：

```bash
diff -u "$HOME/.claude/skills/sglang-triage/SKILL.md" \
  "$SGLANG_KB_ROOT/skills/sglang-triage/SKILL.md"

diff -u "$HOME/.claude/skills/sglang-kb-record/SKILL.md" \
  "$SGLANG_KB_ROOT/skills/sglang-kb-record/SKILL.md"
```

不希望使用符号链接时，也可以手动复制两个目录；每次 `git pull` 后需要重新同步。安装或更新后，新 Skill 内容可能要到下一次调用或新会话才生效。

## 更新规则

`sglang-kb-record` 每次沉淀知识时都会做一次轻量 Skill 复盘：

- 如果当前案例暴露出具体、可复现的流程缺陷，可以把 Skill 修改加入同一份草稿和 diff；
- 用户确认后，知识和 Skill 规范副本进入同一个本地 Git commit；
- commit 后再同步 `$HOME/.claude/skills/` 安装副本；
- 如果安装副本存在额外本地定制，则不自动覆盖，只报告差异；
- 没有具体改进依据时不修改 Skill。
