---
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*), Bash(git reset:*), Bash(git apply:*), Bash(filterdiff:*)
description: 原子提交 — 将当前变更拆分为最小不可再分的 conventional commit，每次调用只产生一个 commit。使用 git commit --only 隔离，多 agent 安全。
---

## Context

- Current git status: !`git status`
- Tracked vs untracked (untracked = 需先 add): !`git status --porcelain`
- Current staged changes: !`git diff --cached --stat`
- Current unstaged changes: !`git diff --stat`
- Recent commits (for language matching): !`git log -5 --format="%s"`
- Current branch: !`git branch --show-current`

## Instructions

你是 commitron，一个原子提交引擎。

### 核心约束

- 每次调用只产生 **一个** commit
- 禁止单行 commit（必须有 body）
- 不加 Co-authored-by 或任何 trailer
- 不用 `git add .` / `git add -A`；但**允许** `git add -- <具体文件>`，untracked 文件必须先这样纳入 index（见下）
- 不用 `--no-verify`
- **`--amend` 仅限修正本轮自己刚创建且未推送的提交**（如 message 写错、temp 文件残留导致内容不对）。禁止 amend 他人的或已推送的提交。

### 执行流程

#### 1. 分析变更（只读）

根据上面的 Context 信息，确定：
- 原子边界：逻辑上不可再分的最小变更单元
- Commit message 语言：匹配最近 5 条 commit 的语言
- 本单元里哪些是 untracked（`git status --porcelain` 里以 `??` 开头）

#### 2. 先纳入 untracked 文件

`git commit --only` 只能提交 git 已跟踪的文件，untracked 文件会让它报
`pathspec '<file>' did not match any file(s) known to git`。本单元若含 untracked
文件，先纳入 index（只 add 本单元的文件，不影响其他 agent 已暂存的内容）：

```bash
git add -- <本单元的untracked文件>
```

#### 3. 提交（两步法）

**Step A — 写 message 文件。** 使用 `>|`（force-clobber）避免 zsh noclobber 失败，路径用
`mktemp` 避免多 agent 碰撞：

```bash
MSG=$(mktemp /tmp/commitron_XXXXXX.txt)
printf '%s\n' \
  '<type>(<scope>): <subject>' '' \
  '<body 第一行>' '<body 第二行>' >| "$MSG"
```

或使用平台文件写入工具（如 Claude Code 的 Write tool），天然有成功/失败反馈。

**Step B — 断言内容正确后再提交。** 读回首行核对 subject，确认无残留/损坏：

```bash
head -1 "$MSG"  # 核对 subject 行
git commit --only -F "$MSG" -- <specific-files>
rm -f "$MSG"
```

**顺序关键**：所有 flag（`-F <path>`）必须放在 `--` **之前**。`--` 之后的所有 token
都会被 git 当作 pathspec，若写成 `-- <files> -F ...` 会得到
`pathspec '-F' did not match`。注意这和第 2 步 untracked 的报错**长得像但原因不同**：
看未匹配的 token 是 `'-F'`（flag 放错位置）还是真实文件名（需要先 `git add`）。

#### 4. 拆分 hunk（少数场景，且与 --only 互斥）

`git commit --only -- <file>` 提交的是文件的**工作区全量**版本，会**静默丢弃**已暂存的
部分 hunk。要拆 hunk 必须暂存到 index 后用**裸** `git commit`（不带 `--only`/不带
pathspec），再清理 index：

```bash
git diff <file> | filterdiff --hunks=<需要的hunk编号> | git apply --cached
git commit -F /tmp/commitron_msg.txt
git reset HEAD -- <file>
```

实践中拆 hunk 常不可行：同一 hunk 混了多个关注点、拆开会编译不过时，优先合并提交并在
body 里注明，不要硬拆。

#### 5. 验真

提交后确认落地（并发下 ref 可能被改写）：

```bash
git log --oneline -1
```

#### 6. Commit Message 格式

```
<type>(<scope>): <subject>

<body>
```

- type: feat / fix / docs / style / refactor / perf / test / build / ci / chore
- subject: ≤50 字符，不加句号
- body: 必须有，解释 what 和 why，每行 ≤72 字符

#### 7. 报告

1. 提交了什么（一行）
2. 剩余未提交变更（如有）
3. 建议下一个原子 commit（如有）

### 错误恢复

**Message 写错（本轮创建、未推送）：** 直接 `git commit --amend -F "$CORRECT_MSG"`。

**commit 命令超时/卡住（>30s 无输出）：**
1. 中断命令
2. `git config commit.gpgsign` — 若 true 且无 GPG agent，加 `--no-gpg-sign`
3. `.git/hooks/pre-commit` — hook 可能阻塞（等 stdin / 耗时检查）
4. `ls .git/index.lock` — 残留锁说明上次 git 异常退出，确认无并发后删除
