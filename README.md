# Commitron

Atomic commit engine for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and Codex. Conventional commits, zero index conflicts.

## Features

- **One commit per call** — refuses to batch multiple logical changes
- **Batch mode** — `commit-all` splits all changes into multiple atomic commits automatically
- **Conventional Commits** — `type(scope): subject` + mandatory body
- **Strict isolation** — uses `git commit --only` to avoid shared index staging conflicts
- **Language matching** — auto-detects commit history language (中文/English/etc.)
- **No co-author trailers** — clean commit history
- **Selective staging** — never uses `git add .`

## Install

### Claude Code

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

### Codex

Install Commitron as a local Codex skill:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/SakuraPuare/commitron ~/.codex/skills/commitron
```

If you already cloned the repository elsewhere, link it instead:

```bash
mkdir -p ~/.codex/skills
ln -s /path/to/commitron ~/.codex/skills/commitron
```

## Usage

In Claude Code or Codex, just say:

- "提交" / "commit" / "原子提交" — single atomic commit
- "全部提交" / "commit all" — split all changes into multiple atomic commits automatically

In Codex, you can also invoke the skill explicitly:

```text
Use $commitron to commit the current changes.
Use $commitron to commit all changes atomically.
```

Commitron will analyze your changes, determine the smallest atomic units, commit only the relevant files, and produce properly formatted messages.

## How It Works

The key innovation is using `git commit --only` for isolated commits. This means:

1. Commitron does not need broad shared-index staging
2. Multiple agents can operate on different files with fewer index conflicts
3. The only serialization point is git's built-in ref lock (atomic by design)

## License

MIT
