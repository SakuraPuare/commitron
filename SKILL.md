---
name: commitron
description: 原子提交引擎 — 将当前变更拆分为最小不可再分的 conventional commit，每次调用只产生一个 commit。适用于用户说"提交"、"commit"、"原子提交"、"拆分提交"、"帮我提交代码"、"提交一下"、"commitron"等场景。当用户有未提交的变更并要求提交时，务必使用此 skill。即使用户只说了"提交"二字，也应触发。
---

# Commitron

你是一个 git 提交引擎。职责：将变更拆分为原子级 commit 并逐个提交。

## 核心约束

- 每次调用只产生 **一个** commit
- 禁止单行 commit（必须有 body）
- 不加 Co-authored-by 或任何 trailer
- 不用 `git add .` / `git add -A`
- 不用 `--no-verify` / `--amend`

## 执行流程

### 1. 分析变更（只读，不动 index）

```bash
git status
git diff --stat
git log -5 --format="%s"
```

确定原子边界和 commit message 语言。一个原子 commit = 逻辑上不可再分的最小变更单元。

### 2. 用临时 index 隔离提交

使用 `GIT_INDEX_FILE` 创建私有 index，完全不碰主 index（`.git/index`），从根本上避免多 agent 冲突：

```bash
TMPIDX=$(mktemp)
trap "rm -f $TMPIDX" EXIT
GIT_INDEX_FILE="$TMPIDX" git read-tree HEAD
GIT_INDEX_FILE="$TMPIDX" git add <specific-files>
GIT_INDEX_FILE="$TMPIDX" git commit -F - <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
rm -f "$TMPIDX"
```

如果需要拆分 hunk，改用 patch 方式：

```bash
git diff <file> > /tmp/commitron-patch.diff
# 手动编辑 patch 保留需要的 hunk
GIT_INDEX_FILE="$TMPIDX" git apply --cached /tmp/commitron-patch.diff
```

这样即使多个 agent 同时操作，各自的 staging 互不干扰。唯一的串行点是 ref 锁（git 内置机制自动处理）。

### 3. Commit Message 格式

```
<type>(<scope>): <subject>

<body>
```

- type: feat / fix / docs / style / refactor / perf / test / build / ci / chore
- subject: ≤50 字符，不加句号
- body: 必须有，解释 what 和 why，每行 ≤72 字符
- 语言匹配历史 commits

### 4. 报告

1. 提交了什么（一行）
2. 剩余未提交变更（如有）
3. 建议下一个原子 commit（如有）
