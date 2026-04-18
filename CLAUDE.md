# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
# Install dependencies
bun install

# Run CLI directly during development
bun run src/index.ts search docker
bun run src/index.ts list --here

# Run tests
bun test

# Run tests with coverage
bun test --coverage

# Build standalone binary
bun run build
# Creates ./deja executable

# Update version
# Edit VERSION constant in src/index.ts and version in package.json
```

## Architecture

**Data Flow**: Session files → Sync → SQLite DB → Search/List

```
~/.claude/projects/*.jsonl  →  src/sync.ts  →  ~/.cc-dejavu/history.db
                                                      ↓
                                          src/commands/search.ts
                                          src/commands/list.ts
                                                      ↓
                                          src/format.ts → terminal
```

**Core Modules**:

| Module | Purpose |
|--------|---------|
| `src/index.ts` | CLI entry point, command routing, help text |
| `src/cli.ts` | Argument parsing (`--regex`, `--cwd`, `--here`, `--limit`, `--sort`) |
| `src/sync.ts` | Parse JSONL session files, extract Bash tool_use/tool_result pairs |
| `src/db.ts` | SQLite operations, LIKE search queries, frecency aggregation |
| `src/frecency.ts` | Ranking algorithm: frequency × recency weights |
| `src/format.ts` | Terminal output with status indicators and match highlighting |
| `src/commands/*.ts` | Individual command implementations (search, list, sync, onboard) |

**Key Design Decisions**:

1. **LIKE search, not FTS5** - FTS5 requires native compilation that fails in npm installs. LIKE queries + frecency work well for command history sizes (<100k commands).

2. **Frecency ranking** - Combines frequency (log10 scale) and recency (time-decay: 4h=100, 24h=70, 7d=50, 30d=30). Formula: `score = (1 + log10(frequency)) × recencyWeight`

3. **Incremental sync** - Tracks byte offsets in `indexed_files` table. Subsequent syncs only parse new content appended to session files.

4. **Auto-sync** - `search` and `list` automatically sync before querying. Use `--no-sync` to skip for faster queries.

5. **Local-only, read-only** - Never modifies Claude session files. All data stays local at `~/.cc-dejavu/`.

**Data Model** (in SQLite):

- `commands` table: `id`, `tool_use_id`, `command`, `description`, `cwd`, `stdout`, `stderr`, `is_error`, `timestamp`, `session_id`
- `indexed_files` table: `file_path`, `last_byte_offset`, `last_modified`

**Session Parsing** (`src/sync.ts`):

JSONL files contain alternating `assistant` (tool_use) and `user` (tool_result) messages. The parser:
1. Collects pending tool_use entries with Bash commands
2. Matches them with corresponding tool_result entries
3. Extracts stdout/stderr from tool_result content

<mental>
Use `mental` to maintain the mental model as you work on the codebase.

<commands>
- `mental add domain <name>` - Add a domain (core business concept)
- `mental add capability <name>` - Add a capability (what the system does)
- `mental add aspect <name>` - Add an aspect (cross-cutting concern)
- `mental add decision <what> --why <reason>` - Record a decision
- `mental update <type> <name|id>` - Update an entity (--desc, --files, --relates-to, etc.)
- `mental supersede decision <id>` - Replace a decision with a new one (keeps history)
- `mental show` - Display the current model
- `mental view` - Open interactive visualization
</commands>

<when-to-use>
- After adding a new feature or module
- After making an architectural decision
- When introducing a new domain concept
- When adding cross-cutting concerns (auth, logging, caching)
- When the user asks to update the mental model
</when-to-use>

<guidelines>
- Domains are nouns (User, Order, Payment)
- Capabilities are verbs (Checkout, ProcessPayment, SendNotification)
- Aspects apply across capabilities (Auth, Validation, RateLimit)
- Decisions capture the "why" behind choices
- Link decisions to related entities with --relates-to
- Attach documentation with --docs (local paths or URLs)
</guidelines>
</mental>