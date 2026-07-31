---
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*), Bash(git reset:*), Bash(git apply:*), Bash(filterdiff:*)
description: 批量原子提交 — 将所有当前变更自动拆分为多个最小不可再分的 conventional commit，依次提交。使用 git commit --only 隔离，多 agent 安全。
---

## Context

- Current git status: !`git status`
- Tracked vs untracked (untracked = 需先 add): !`git status --porcelain`
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
- 不用 `git add .` / `git add -A`；但**允许** `git add -- <具体文件>`，untracked 文件必须先这样纳入 index（见下）
- 不用 `--no-verify`
- **`--amend` 仅限修正本轮自己刚创建且未推送的提交**（如 message 写错）。禁止 amend 他人的或已推送的提交。

### 执行流程

#### 1. 分析全部变更（只读）

根据上面的 Context 信息，确定：
- 所有变更的原子边界：将全部 staged + unstaged 变更划分为逻辑上不可再分的最小变更单元
- 每个原子单元的提交顺序（依赖关系优先）
- Commit message 语言：匹配最近 5 条 commit 的语言
- 哪些是 untracked（`git status --porcelain` 里以 `??` 开头），提交前需先 add

#### 2. 依次提交每个原子单元

对每个原子单元：

**a. 先纳入 untracked 文件。** `git commit --only` 只能提交已跟踪文件，untracked
文件会让它报 `pathspec '<file>' did not match any file(s) known to git`。本单元若含
untracked 文件，先 `git add -- <本单元的untracked文件>`（只 add 本单元，不影响其他
agent 已暂存内容，隔离性保留）。

**b. 提交（两步法）。** 先写 message 文件，断言内容正确，再 commit。使用 `>|`（force-clobber）
避免 zsh noclobber 失败，路径用 `mktemp` 避免多 agent 碰撞：

```bash
MSG=$(mktemp /tmp/commitron_XXXXXX.txt)
printf '%s\n' \
  '<type>(<scope>): <subject>' '' \
  '<body 第一行>' '<body 第二行>' >| "$MSG"
head -1 "$MSG"  # 断言 subject 行正确
git commit --only -F "$MSG" -- <specific-files>
rm -f "$MSG"
```

或使用平台文件写入工具（如 Claude Code 的 Write tool）写 message 文件，天然有成功/失败
反馈，可省略 head -1 核对。

**顺序关键**：所有 flag（`-F <path>`）必须放在 `--` **之前**。`--` 之后的 token 都会被
当作 pathspec，若写成 `-- <files> -F ...` 会得到 `pathspec '-F' did not match`。注意这和
a 步 untracked 的报错**长得像但原因不同**：看未匹配 token 是 `'-F'`（flag 放错）还是真实
文件名（需先 `git add`）。

**c. 验真。** 提交后 `git log --oneline -1` 确认落地（并发下 ref 可能被改写）。

拆分 hunk（少数场景，且与 `--only` 互斥）：`git commit --only -- <file>` 提交的是工作区
全量版本，会**静默丢弃**已暂存的部分 hunk。要拆 hunk 必须暂存到 index 后用**裸**
`git commit`（不带 `--only`/不带 pathspec）：

```bash
git diff <file> | filterdiff --hunks=<需要的hunk编号> | git apply --cached
git commit -F /tmp/commitron_msg.txt
git reset HEAD -- <file>
```

实践中同一 hunk 混多个关注点、拆开会编译不过时，优先合并提交并在 body 注明，不要硬拆。

### 并发

`--only` 隔离的是**文件集**，但 `.git/index.lock` 是全局单锁。并行 agent 同时提交会撞
`fatal: Unable to create '.../.git/index.lock': File exists`。多 agent 场景需串行化提交，
或撞锁时短暂退避重试。`LF/CRLF will be replaced` 是 autocrlf 噪音，可忽略。

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

### 错误恢复

**Message 写错（本轮创建、未推送）：** 直接 `git commit --amend -F "$CORRECT_MSG"`。

**commit 命令超时/卡住（>30s 无输出）：**
1. 中断命令
2. `git config commit.gpgsign` — 若 true 且无 GPG agent，加 `--no-gpg-sign`
3. `.git/hooks/pre-commit` — hook 可能阻塞
4. `ls .git/index.lock` — 残留锁说明上次 git 异常退出，确认无并发后删除
