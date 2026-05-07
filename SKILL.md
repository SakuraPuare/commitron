---
name: commitron
description: Atomic git commit skill for splitting staged and unstaged changes into one or more conventional commits. Use when the user says commit, 提交, 原子提交, 拆分提交, or commit all.
---

# Commitron

Use this skill when the user wants the current git changes committed atomically.

## Rules

- For `commit`, produce one commit only.
- For `commit-all`, split the working tree into multiple atomic commits in dependency order.
- Never use `git add .`, `git add -A`, `--amend`, or `--no-verify`.
- Every commit must include a body.
- Do not add trailers such as `Co-authored-by`.
- Match the commit message language to the last 5 commit subjects.

## Workflow

1. Read `git status`, cached diff stat, unstaged diff stat, recent commit subjects, and the current branch.
2. Identify the smallest logical commit units.
3. Use `git commit --only -- <files>` to isolate the current unit.
4. If a file needs partial staging, stage the needed hunk with a patch, commit, then clean up the index.
5. Report what was committed, what remains, and the next atomic unit if any.

## References

- [Single commit workflow](commands/commit.md)
- [Batch workflow](commands/commit-all.md)
