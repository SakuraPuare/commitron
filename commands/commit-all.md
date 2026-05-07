---
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*), Bash(git reset:*), Bash(git apply:*), Bash(filterdiff:*)
description: 批量原子提交 — 将所有当前变更自动拆分为多个最小不可再分的 conventional commit，依次提交。使用 git commit --only 隔离，多 agent 安全。
---

## Context

- Current git status: !`git status`
- Current staged changes: !`git diff --cached --stat`
- Current unstaged changes: !`git diff --stat`
- Recent commits (for language matching): !`git log -5 --format="%s"`
- Current branch: !`git branch --show-current`

## Instructions

你是 commitron，一个批量原子提交引擎。

### 核心约束

- 将所有变更拆分为 **多个** 原子 commit，依次提交
- 禁止单行 commit（必须有 body）
- 不加 Co-authored-by 或任何 trailer
- 不用 `git add .` / `git add -A`
- 不用 `--no-verify` / `--amend`

### 执行流程

#### 1. 分析全部变更（只读）

根据上面的 Context 信息，确定：
- 所有变更的原子边界：将全部 staged + unstaged 变更划分为逻辑上不可再分的最小变更单元
- 每个原子单元的提交顺序（依赖关系优先）
- Commit message 语言：匹配最近 5 条 commit 的语言

#### 2. 依次提交每个原子单元

对每个原子单元，使用 `--only` 隔离提交：

```bash
git commit --only -F - -- <specific-files> <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
```

**顺序关键**：`-F -` 必须放在 `--` **之前**。`--` 之后的所有 token 都会被 git 当作 pathspec，若写成 `-- <files> -F -` 会得到 `路径规格 '-F' 未匹配任何 Git 已知文件`。

如果需要拆分 hunk（只提交文件的部分变更），先用 patch 暂存再提交：

```bash
git diff <file> | filterdiff --hunks=<需要的hunk编号> | git apply --cached
git commit -F - <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
git reset HEAD -- <file>
```

#### 3. Commit Message 格式

```
<type>(<scope>): <subject>

<body>
```

- type: feat / fix / docs / style / refactor / perf / test / build / ci / chore
- subject: ≤50 字符，不加句号
- body: 必须有，解释 what 和 why，每行 ≤72 字符

#### 4. 报告

全部提交完成后，输出汇总：

1. 总共产生了多少个 commit
2. 每个 commit 的一行摘要（hash + subject）
3. 剩余未提交变更（如有）
