<p align="center">
  <img src="assets/hero.gif" alt="deja">
</p>

# deja

[![CI](https://github.com/Michaelliv/cc-dejavu/actions/workflows/ci.yml/badge.svg)](https://github.com/Michaelliv/cc-dejavu/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/Michaelliv/cc-dejavu/graph/badge.svg)](https://codecov.io/gh/Michaelliv/cc-dejavu)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Bun](https://img.shields.io/badge/Bun-%23000000.svg?logo=bun&logoColor=white)](https://bun.sh)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)

Search and browse your Claude Code bash command history and conversation content.

```
"What was that docker command I ran yesterday?"
"What did we discuss about MCP configuration?"
```

```bash
$ deja search docker --limit 4

[ok] docker build --no-cache --platform linux/amd64 -t ghcr.io/user/api-service:latest .
     Rebuild without cache for production
     12/30/2025, 12:46 AM | ~/projects/api-service

[ok] docker build -t api-service:test .
     Build test image
     12/30/2025, 12:45 AM | ~/projects/api-service
```

```bash
$ deja chat search "mcp配置"

[text]
  已删除。Playwright MCP 配置和进程都已清理干净。
  4/18/2026, 11:50:36 PM | ~/Documents/zwork/zoeskills | session: f984ae62...

[text]
  帮我把playwright mcp先删掉吧
  4/18/2026, 11:50:06 PM | ~/Documents/zwork/zoeskills | session: f984ae62...

2 messages
```

Every bash command and conversation message Claude Code runs is logged in session files. `deja` indexes them into a searchable database so you can find commands, review discussions, and avoid repeating mistakes.

## Features

- **Smart ranking** - Frecency (frequency + recency) ranking so the most useful commands appear first
- **LIKE search** - Simple substring matching with SQLite LIKE queries
- **Conversation search** - Search assistant text, thinking blocks, and tool interactions
- **Match highlighting** - Search patterns are highlighted in yellow in the output
- **Automatic sync** - History is automatically synced before each search
- **Candidate extraction** - Extract high-value content for knowledge ingestion

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/Michaelliv/cc-dejavu/main/install.sh | bash
```

Or with Bun:
```bash
bun add -g cc-dejavu
```

<details>
<summary>Other options</summary>

**Manual download**: Get binaries from [releases](https://github.com/Michaelliv/cc-dejavu/releases)

**From source**:
```bash
git clone https://github.com/Michaelliv/cc-dejavu
cd cc-dejavu
bun install && bun run build
```
</details>

## Usage

### Command Search

```bash
# Search for commands containing "docker"
deja search docker

# Use regex patterns
deja search "git commit.*fix" --regex

# Filter by project directory
deja search npm --cwd /projects/myapp

# Filter by current directory
deja search npm --here

# Sort by time instead of frecency
deja search npm --sort time

# List recent commands
deja list
deja list --limit 50

# List commands from current project only
deja list --here

# Manually sync (usually automatic)
deja sync
deja sync --force  # Re-index everything
```

### Conversation Search

```bash
# Search conversation content
deja chat search "部署"

# Search only thinking blocks
deja chat search "设计" --type thinking

# Search all content types
deja chat search "error" --type all

# View a full session timeline
deja chat session <session_id>
```

### Candidate Extraction

```bash
# Extract high-value content for knowledge ingestion
deja ingest --candidates

# Filter by session
deja ingest --candidates --session <session_id>

# Set minimum value score threshold
deja ingest --candidates --min-score 70

# Output as JSON (for scripts/hooks)
deja ingest --candidates --format json
```

## Commands

### `deja search <pattern>`

Search command history by substring or regex.

| Flag | Description |
|------|-------------|
| `--regex`, `-r` | Treat pattern as regular expression |
| `--cwd <path>` | Filter by working directory |
| `--here`, `-H` | Filter by current directory |
| `--limit`, `-n <N>` | Limit number of results |
| `--sort <mode>` | Sort by: `frecency` (default), `time` |
| `--no-sync` | Skip auto-sync before searching |

### `deja list`

Show recent commands, sorted by frecency by default.

| Flag | Description |
|------|-------------|
| `--limit`, `-n <N>` | Number of commands (default: 20) |
| `--here`, `-H` | Filter by current directory |
| `--sort <mode>` | Sort by: `frecency` (default), `time` |
| `--no-sync` | Skip auto-sync before listing |

### `deja chat search <pattern>`

Search conversation content (text, thinking, tool interactions).

| Flag | Description |
|------|-------------|
| `--type <type>` | Filter by: `text`, `thinking`, `all` (default: all) |
| `--cwd <path>` | Filter by working directory |

### `deja chat session <session_id>`

View a complete session timeline with all messages and commands.

### `deja ingest --candidates`

Extract high-value content candidates for knowledge ingestion.

| Flag | Description |
|------|-------------|
| `--session <id>` | Filter by session ID |
| `--min-score <N>` | Minimum value score (default: 50) |
| `--format <format>` | Output format: `text` (default), `json` |

### `deja sync`

Index new commands and messages from Claude Code sessions.

| Flag | Description |
|------|-------------|
| `--force`, `-f` | Re-index all sessions from scratch |

## Ranking Algorithm

Commands are ranked by **frecency** (frequency + recency):

```
score = (1 + log10(frequency)) × recencyWeight
```

- **Frequency**: Uses logarithmic scaling so popular commands don't dominate
- **Recency**: Time-decay weights (4h=100, 24h=70, 7d=50, 30d=30)

This means commands you run often AND recently will appear at the top.

Use `--sort time` to revert to simple timestamp ordering.

## Value Scoring

Candidates are scored (0-100) based on:

- **Content type**: Thinking blocks score higher (70 base)
- **Word count**: Longer content scores higher (up to +30)
- **Keyword patterns**: Contains "solve|fix|implement|design|error|important" (+15)

## How It Works

Claude Code stores conversation data in `~/.claude/projects/`. Each session is a JSONL file containing messages, tool calls, and results.

`deja` scans these files, extracts:
- **Bash tool invocations** → `commands` table
- **Assistant text/thinking** → `messages` table
- **High-value content** → `candidates` table

Data is stored in a local SQLite database at `~/.cc-dejavu/history.db`.

**Auto-sync**: By default, `search` and `list` automatically sync before returning results. Use `--no-sync` to skip this if you want faster queries.

**Privacy**: `deja` is read-only and local-only. It reads Claude's session files but never modifies them. No data is sent anywhere.

## Data Model

### Commands Table

| Field | Description |
|-------|-------------|
| `command` | The bash command that was executed |
| `description` | What Claude said it does |
| `cwd` | Working directory when command ran |
| `timestamp` | When the command was executed |
| `is_error` | Whether the command failed |
| `stdout` | Command output |
| `stderr` | Error output |
| `session_id` | Which Claude session ran this command |

### Messages Table

| Field | Description |
|-------|-------------|
| `uuid` | Unique message identifier |
| `session_id` | Which Claude session |
| `type` | `user` or `assistant` |
| `content_type` | `text`, `thinking`, `tool_use`, `tool_result` |
| `content` | The message content |
| `timestamp` | When the message was sent |
| `cwd` | Working directory context |

### Candidates Table

| Field | Description |
|-------|-------------|
| `session_id` | Source session |
| `content_type` | Type of content |
| `content` | The candidate content |
| `value_score` | Quality score (0-100) |
| `timestamp` | When extracted |

## For AI Agents

Run `deja onboard` to add a section to `~/.claude/CLAUDE.md` so Claude knows how to search its own history:

```bash
deja onboard
```

This adds instructions for using `deja search`, `deja list`, and `deja chat search`.

### When to use `deja`

- User asks "what was that command I/you ran?"
- User wants to find commands from a previous session
- User asks "what did we discuss about X?"
- User wants to recall decisions or thinking from past sessions

### When NOT to use `deja`

- Finding files by name -> use `Glob`
- Searching file contents -> use `Grep`
- Checking recent conversation context -> already in your context

## SessionEnd Hook Integration

The `deja ingest --candidates --format json` output can be used in Claude Code's SessionEnd hook to extract valuable content for knowledge systems.

Example hook script:

```bash
#!/bin/bash
SESSION_ID="${CLAUDE_SESSION_ID:-}"
deja ingest --candidates --session "$SESSION_ID" --min-score 50 --format json
```

Output format:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "DejaCandidates",
    "candidates": [
      {
        "type": "thinking",
        "content": "...",
        "score": 75,
        "session": "..."
      }
    ]
  }
}
```

## Development

```bash
# Install dependencies
bun install

# Run directly
bun run src/index.ts search docker
bun run src/index.ts chat search "部署"

# Run tests
bun test

# Run tests with coverage
bun test --coverage

# Build binary
bun run build
```

## About the name

`deja` - short for deja vu, "already seen." It shows you commands and conversations you've already had.

---

MIT License