# rledger - AI Assistant Instructions

## Project Context
This is rledger, a modern plain-text accounting system in Rust. See docs/TECHNICAL_SPEC.md for complete details.

## Key Rules
1. Use workspace dependencies: `{ workspace = true }`
2. Use `Decimal` for all monetary amounts, NEVER `f64`
3. Use `thiserror` for library errors, `anyhow` for application errors
4. Document all public APIs with examples
5. Add tests for all new code (target: 80% coverage)
6. Follow Rust 2024 edition idioms
7. Immutable data structures preferred
8. Check TECHNICAL_SPEC.md before making architectural decisions

## File Structure
- `crates/rledger-common`: Shared types (AccountName, Amount, Transaction)
- `crates/rledger-core`: Parser (nom) and validation logic
- `crates/rledger-storage`: SQLite cache layer
- `crates/rledger-cli`: CLI interface (clap)

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

## When in doubt

Check docs/TECHNICAL_SPEC.md
Prioritize correctness over performance
Ask for clarification on ambiguous requirements
