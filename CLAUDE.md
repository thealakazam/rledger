# rledger - AI Assistant Instructions

## Project Context

rledger is a Rust plaintext accounting tool — Beancount-compatible syntax, SQLite-backed incremental cache, lot tracking engine, mobile FFI target.

**Key docs (read before making architectural decisions):**
- `docs/implementation-plan.md` — 15-phase plan, effort estimates, dependency graph, critical files
- `docs/syntax-specification.md` — full grammar, all directive types, token reference
- `docs/pta-consolidated-decisions.md` — why Beancount format, SQLite hybrid arch, booking pipeline design

**Implementation status:** Phase 0 in progress. All crates are empty stubs. See `bd ready` for next work.

## Crate Structure

| Crate | Purpose |
|---|---|
| `rledger-common` | Shared types: AccountName, Amount, Transaction, Commodity |
| `rledger-core` | Lexer (logos), parser (nom/recursive descent), lowering, booking engine, validation |
| `rledger-storage` | SQLite cache layer (rusqlite bundled) |
| `rledger-cli` | CLI binary (clap) |

## Key Rules

1. Workspace deps only: `{ workspace = true }` — never add versions in crate Cargo.toml
2. `Decimal` for all monetary amounts — NEVER `f64`
3. `thiserror` for library errors, `anyhow` for application errors
4. Document all public APIs with examples
5. Tests for all new code (target: 80% coverage)
6. Rust 2024 edition idioms
7. Immutable data structures preferred
8. Amounts stored as TEXT in SQLite — never as REAL (precision loss)

## Common Patterns

```rust
// Error handling
use anyhow::{Context, Result};
let data = load_data().context("Failed to load")?;

// Decimal for money
use rust_decimal::Decimal;
let amount = Decimal::from_str("10.50")?;

// Dates
use chrono::NaiveDate;
let date = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();
```

## MCP Tool Usage

### Git MCP
Use git MCP server tools for ALL git operations (status, diff, log, commit, push, branch) when a git MCP server is connected. Prefer MCP tools over `Bash` for git. Fall back to Bash only if no git MCP is available.

### Beads (Task Tracking)
**Always use `bd` (beads) for task tracking.** Never use TodoWrite, TaskCreate, or markdown TODO lists. Beads is the single source of truth for all work items.

Key commands:
```bash
bd ready                    # What's unblocked and available
bd show <id>                # Full issue detail
bd update <id> --claim      # Claim before starting
bd close <id>               # Mark complete
bd remember "insight"       # Persist knowledge across sessions
```

## When in doubt

- Read `docs/implementation-plan.md` for the phase you're working on
- Prioritize correctness over performance
- Ask for clarification on ambiguous requirements


<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
