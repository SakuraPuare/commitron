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
- Never use `--amend` or `--no-verify`.
- Every commit must include a body.
- Do not add trailers such as `Co-authored-by`.
- Match the commit message language to the last 5 commit subjects.

## Workflow

1. Read `git status --porcelain` (this is the only view that lists untracked files — cached/unstaged diff stat do not), cached diff stat, unstaged diff stat, recent commit subjects, and the current branch.
2. Identify the smallest logical commit units.
3. **Stage untracked files first.** `git commit --only` can only commit files git already tracks; an untracked file makes it fail with `pathspec '<file>' did not match any file(s) known to git`. For any untracked file in the current unit, run `git add -- <those files>` first. This preserves isolation — files staged by other agents stay in the index untouched.
4. Isolate and commit the current unit. Write the message to a temp file and pass it with `-F <path>`, avoiding the fragile `-F -` + heredoc form:

   ```bash
   printf '%s\n' \
     '<type>(<scope>): <subject>' '' \
     '<body line 1>' '<body line 2>' > /tmp/commitron_msg.txt
   git commit --only -F /tmp/commitron_msg.txt -- <files>
   ```

   All flags (`-F <path>`) must come **before** `--`; anything after `--` is a pathspec. Writing `-- <files> -F ...` fails with `pathspec '-F' did not match`. Note this is a **different** error from the untracked-file case in step 3, though the message looks similar — check whether the unmatched token is `'-F'` (flag misplaced) or a real filename (needs `git add` first).
5. **Never combine `--only` with partial-hunk staging.** `git commit --only -- <file>` commits the file's **working-tree** version and silently discards any partially-staged hunk. To split hunks, stage the needed hunk to the index (`git diff <file> | filterdiff --hunks=<n> | git apply --cached`), then commit the index with a **bare** `git commit -F <path>` (no `--only`, no pathspec), and `git reset HEAD -- <file>` to clean up. In practice, hunk-splitting is rarely viable — when one hunk mixes concerns that cannot compile apart, prefer committing them together and noting it in the body.
6. **Verify each commit landed.** After committing, run `git log --oneline -1` (or `git show --stat HEAD`) to confirm the commit exists with the expected files — under concurrent agents a ref can shift.
7. Report what was committed, what remains, and the next atomic unit if any.

## Concurrency

`--only` isolates the **file set**, but `.git/index.lock` is a single global lock. Parallel agents committing at the same time can collide with `fatal: Unable to create '.../.git/index.lock': File exists`. Serialize commits across agents, or retry with a short backoff when the lock is held.

## Environment noise

`warning: ... LF will be replaced by CRLF` (and the reverse) is harmless autocrlf noise. Ignore it; it does not affect what gets committed.

## References

- [Single commit workflow](commands/commit.md)
- [Batch workflow](commands/commit-all.md)
