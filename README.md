# Commitron

Atomic commit engine for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). One invocation, one conventional commit, zero index conflicts.

## Features

- **One commit per call** — refuses to batch multiple logical changes
- **Conventional Commits** — `type(scope): subject` + mandatory body
- **Strict isolation** — uses `GIT_INDEX_FILE` for private staging, safe with parallel agents
- **Language matching** — auto-detects commit history language (中文/English/etc.)
- **No co-author trailers** — clean commit history
- **Selective staging** — never uses `git add .`

## Install

```bash
claude plugins marketplace add SakuraPuare/commitron
claude plugins install commitron
```

## Usage

In Claude Code, just say:

- "提交"
- "commit"
- "原子提交"
- "帮我提交代码"

Commitron will analyze your changes, determine the smallest atomic unit, stage only the relevant files into an isolated index, and commit with a properly formatted message.

## How It Works

The key innovation is using `GIT_INDEX_FILE` to create a private temporary index per agent invocation. This means:

1. The shared `.git/index` is never touched during staging
2. Multiple agents can operate on the same repo simultaneously without conflicts
3. The only serialization point is git's built-in ref lock (atomic by design)

## License

MIT
