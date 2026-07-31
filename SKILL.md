---
name: commitron
description: Atomic git commit skill for splitting staged and unstaged changes into one or more conventional commits. Use when the user says commit, 提交, 原子提交, 拆分提交, or commit all.
---

# Commitron

Use this skill when the user wants the current git changes committed atomically.

## Rules

- For `commit`, produce one commit only.
- For `commit-all`, split the working tree into multiple atomic commits in dependency order.
- Never use `git add .` or `git add -A`. Staging **specific** files with `git add -- <files>` is allowed and required for untracked files (see below).
- Never use `--no-verify`.
- **`--amend` 仅限修正本轮自己刚创建且未推送的提交**（如 message 写错、工具故障导致内容不对）。禁止 amend 他人的或已推送的提交。
- Every commit must include a body.
- Do not add trailers such as `Co-authored-by`.
- Match the commit message language to the last 5 commit subjects.

## Workflow

1. Read `git status --porcelain` (this is the only view that lists untracked files — cached/unstaged diff stat do not), cached diff stat, unstaged diff stat, recent commit subjects, and the current branch.
2. Identify the smallest logical commit units.
3. **Stage untracked files first.** `git commit --only` can only commit files git already tracks; an untracked file makes it fail with `pathspec '<file>' did not match any file(s) known to git`. For any untracked file in the current unit, run `git add -- <those files>` first. This preserves isolation — files staged by other agents stay in the index untouched.
4. **Isolate and commit the current unit (两步法).** 写 message 和提交必须分开执行，中间加断言：

   **Step A — 写 message 文件：**
   使用 `>|`（force-clobber）而非 `>`，或用平台的文件写入工具（如 Claude Code 的 Write tool）。路径使用 `mktemp` 避免碰撞：

   ```bash
   MSG=$(mktemp /tmp/commitron_XXXXXX.txt)
   printf '%s\n' \
     '<type>(<scope>): <subject>' '' \
     '<body line 1>' '<body line 2>' >| "$MSG"
   ```

   **Step B — 断言内容正确后再提交：**
   读回文件首行，确认与预期 subject 一致（防 noclobber 静默失败、编码损坏等）：

   ```bash
   head -1 "$MSG"  # 肉眼/脚本核对 subject 行
   git commit --only -F "$MSG" -- <files>
   rm -f "$MSG"
   ```

   若使用平台 Write tool 写 message，则天然有写入成功/失败的反馈，可省略 head -1 核对。

   **Flag 顺序**：所有 flag（`-F <path>`）必须放在 `--` **之前**。`--` 之后的所有 token 都会被当作 pathspec，若写成 `-- <files> -F ...` 会得到 `pathspec '-F' did not match`。注意这和第 3 步 untracked 的报错**长得像但原因不同**——看未匹配 token 是 `'-F'`（flag 放错）还是真实文件名（需先 `git add`）。

5. **Never combine `--only` with partial-hunk staging.** `git commit --only -- <file>` commits the file's **working-tree** version and silently discards any partially-staged hunk. To split hunks, stage the needed hunk to the index (`git diff <file> | filterdiff --hunks=<n> | git apply --cached`), then commit the index with a **bare** `git commit -F <path>` (no `--only`, no pathspec), and `git reset HEAD -- <file>` to clean up. In practice, hunk-splitting is rarely viable — when one hunk mixes concerns that cannot compile apart, prefer committing them together and noting it in the body.
6. **Verify each commit landed.** After committing, run `git log --oneline -1` (or `git show --stat HEAD`) to confirm the commit exists with the expected files — under concurrent agents a ref can shift.
7. **若 commit 命令超时/卡住**（>30s），中断后排查：`git config commit.gpgsign`（GPG 等签名输入）、`.git/hooks/pre-commit`（hook 阻塞）。确认阻塞源后重试或加 `--no-gpg-sign`（仅签名卡住时）。
7. Report what was committed, what remains, and the next atomic unit if any.

## Concurrency

`--only` isolates the **file set**, but `.git/index.lock` is a single global lock. Parallel agents committing at the same time can collide with `fatal: Unable to create '.../.git/index.lock': File exists`. Serialize commits across agents, or retry with a short backoff when the lock is held.

## Error Recovery

**Message 写错/工具故障导致提交内容不对（本轮创建、未推送）：**

```bash
git commit --amend -F "$CORRECT_MSG"
```

仅限修正本轮自己刚创建的提交。若已推送，使用 `git revert` 而非 amend。

**commit 命令卡住（>30s 无输出）：**
1. 中断命令
2. 检查 `git config commit.gpgsign`——若为 true 且无 GPG agent，加 `--no-gpg-sign`
3. 检查 `.git/hooks/pre-commit`——hook 可能在等 stdin 或跑了耗时分析
4. 检查 `ls .git/index.lock`——残留锁文件说明上次 git 进程异常退出，确认无并发后 `rm .git/index.lock`

## Environment noise

`warning: ... LF will be replaced by CRLF` (and the reverse) is harmless autocrlf noise. Ignore it; it does not affect what gets committed.

## References

- [Single commit workflow](commands/commit.md)
- [Batch workflow](commands/commit-all.md)
