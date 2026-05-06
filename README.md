# Commitron

Atomic commit engine for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Conventional commits, zero index conflicts.

## Features

- **One commit per call** — refuses to batch multiple logical changes
- **Batch mode** — `commit-all` splits all changes into multiple atomic commits automatically
- **Conventional Commits** — `type(scope): subject` + mandatory body
- **Strict isolation** — uses `GIT_INDEX_FILE` for private staging, safe with parallel agents
- **Language matching** — auto-detects commit history language (中文/English/etc.)
- **No co-author trailers** — clean commit history
- **Selective staging** — never uses `git add .`

## Install

```bash
# First time: add the marketplace
claude plugins marketplace add SakuraPuare/commitron

# Install
claude plugins install commitron
```

To update later:

```bash
# Refresh marketplace cache first, then update
# (skipping the first step may install an old cached version)
claude plugins marketplace update
claude plugins update commitron@commitron
```

## Usage

In Claude Code, just say:

- "提交" / "commit" / "原子提交" — single atomic commit
- "全部提交" / "commit all" — split all changes into multiple atomic commits automatically

Commitron will analyze your changes, determine the smallest atomic units, stage only the relevant files into an isolated index, and commit with properly formatted messages.

## How It Works

The key innovation is using `GIT_INDEX_FILE` to create a private temporary index per agent invocation. This means:

1. The shared `.git/index` is never touched during staging
2. Multiple agents can operate on the same repo simultaneously without conflicts
3. The only serialization point is git's built-in ref lock (atomic by design)

## License

MIT
