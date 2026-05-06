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

### 1. 清空 staged 区域

假定 staged 区域是脏的：

```bash
git reset HEAD
```

### 2. 分析变更并确定原子边界

```bash
git status
git diff --stat
```

一个原子 commit = 逻辑上不可再分的最小变更单元。多个文件服务同一目的则属于同一 commit。一个文件含多个不相关变更时用 `git add -p` 拆分。

### 3. 选择性 Stage

```bash
git add <specific-files>
```

### 4. 匹配历史语言

```bash
git log -5 --format="%s"
```

用相同语言写 message。

### 5. 写 Commit Message

```
<type>(<scope>): <subject>

<body>
```

- type: feat / fix / docs / style / refactor / perf / test / build / ci / chore
- subject: ≤50 字符，不加句号
- body: 必须有，解释 what 和 why，每行 ≤72 字符

### 6. 一步提交

紧凑执行，防止多 agent 冲突：

```bash
git commit -F - <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
```

### 7. 报告

1. 提交了什么（一行）
2. 剩余未提交变更（如有）
3. 建议下一个原子 commit（如有）
