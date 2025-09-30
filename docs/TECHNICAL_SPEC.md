# rledger - Technical Specification & Architecture Guide

**Version:** 0.1.0 (Phase 1 - Core Engine)  
**Last Updated:** September 2025  
**Status:** Active Development

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Principles](#architecture-principles)
3. [Technology Stack](#technology-stack)
4. [Crate Structure](#crate-structure)
5. [Data Models](#data-models)
6. [Coding Standards](#coding-standards)
7. [Decision Log](#decision-log)
8. [Implementation Guidelines](#implementation-guidelines)

---

## Project Overview

### Vision
A modern, high-performance plain-text accounting system written in Rust that reimagines ledger/hledger with focus on:
- **Performance**: 10-50x faster than existing tools
- **Simplicity**: Intuitive UX without sacrificing power
- **Compatibility**: Drop-in replacement for ledger/hledger workflows
- **Modern**: Built with 2025 best practices

### Target Users
1. **Primary**: Technical prosumers (developers, engineers managing personal/small business finances)
2. **Secondary**: Small business owners with technical aptitude
3. **Tertiary**: Investment tracking enthusiasts

### Core Value Propositions
- Parse 100k transactions in <2 seconds (vs 40s+ for GnuCash)
- Plain-text files with git-friendly diffs
- Built-in lot tracking for investments (missing in hledger/ledger)
- Native support for Indian securities (NSE/BSE/AMFI)
- Memory-safe (Rust vs C/C++)

---

## Architecture Principles

### 1. Layered Architecture

```
┌─────────────────────────────────────┐
│  Presentation Layer                 │
│  (CLI, Web UI, Desktop)             │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│  API Layer (Future)                 │
│  (REST endpoints)                   │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│  Business Logic Layer               │
│  (Accounting rules, validation)     │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│  Data Access Layer                  │
│  (Parser, SQLite, Cache)            │
└─────────────────────────────────────┘
```

### 2. Design Principles

**Separation of Concerns**
- Parsing is separate from validation
- Business logic is separate from storage
- Each crate has a single, clear responsibility

**Immutability**
- Transactions are immutable once written
- Corrections via reversing entries (audit trail)
- Cache invalidation over mutation

**Zero-Copy Where Possible**
- Use string slices in parser
- Avoid unnecessary allocations
- Memory-mapped files for large journals

**Fail Fast**
- Validate at boundaries (parse time, not runtime)
- Comprehensive error types
- No silent failures

**Performance by Default**
- Async I/O for all blocking operations
- Parallel processing for bulk operations (rayon)
- Caching with smart invalidation

---

## Technology Stack

### Language & Tooling
- **Rust**: 1.90+ (MSRV), Edition 2024
- **Cargo**: Workspace with resolver = "2"
- **Compiler**: Latest stable

### Core Dependencies

| Crate | Version | Purpose | Why This Choice |
|-------|---------|---------|-----------------|
| `rust_decimal` | 1.35 | Monetary amounts | Exact decimal arithmetic, no floating point errors |
| `chrono` | 0.4 | Date handling | Industry standard, feature-rich |
| `nom` | 7.1 | Parser combinators | Zero-copy parsing, excellent errors |
| `sqlx` | 0.8 | Database | Async, compile-time query checking |
| `tokio` | 1.40 | Async runtime | Most mature, best ecosystem |
| `clap` | 4.5 | CLI parsing | Derive API, great help generation |
| `thiserror` | 1.0 | Library errors | Ergonomic error types |
| `anyhow` | 1.0 | Application errors | Context propagation |
| `serde` | 1.0 | Serialization | Universal standard |
| `tracing` | 0.1 | Structured logging | Better than log crate |

### Storage
- **Primary**: Plain-text journal files (ledger format)
- **Cache**: SQLite 3.35+ (WAL mode)
- **Future**: Optional PostgreSQL for multi-user

---

## Crate Structure

### `rledger-common`
**Purpose**: Shared types and utilities  
**Exports**: Core data types used across all crates  
**Dependencies**: Minimal (rust_decimal, chrono, thiserror, serde)

**Key Types**:
```rust
pub struct AccountName(String);
pub struct Commodity(String);
pub struct Amount { quantity: Decimal, commodity: Commodity }
pub enum Status { Unmarked, Pending, Cleared }
pub struct Posting { account, amount, cost, balance_assertion, is_virtual }
pub struct Transaction { date, status, code, description, postings, metadata }
pub enum AccountType { Asset, Liability, Equity, Revenue, Expense }
```

**Design Decisions**:
- Use newtype pattern for semantic types (AccountName, Commodity)
- All monetary amounts use `Decimal`, never `f64`
- Transactions are `Clone` but large (use `Arc` when sharing)

---

### `rledger-core`
**Purpose**: Journal parsing and accounting logic  
**Exports**: Parser, validator, accounting engine  
**Dependencies**: rledger-common, nom, anyhow, thiserror

**Modules**:
```
rledger-core/
├── journal/
│   ├── parser.rs        # nom-based parser
│   ├── include.rs       # Handle include directives
│   └── directives.rs    # Account, commodity, price directives
├── account/
│   ├── tree.rs          # Account hierarchy
│   └── types.rs         # Account type inference
├── transaction/
│   └── builder.rs       # Transaction construction helpers
├── validation/
│   ├── balance.rs       # Transaction balancing
│   └── assertions.rs    # Balance assertions
└── error.rs             # Error types
```

**Key Algorithms**:
- **Parser**: Zero-copy nom combinators
- **Balancing**: Group by commodity, sum debits/credits, check within epsilon
- **Auto-balance**: Find omitted posting, calculate amount to balance
- **Assertions**: Check in date order, fail fast on mismatch

**Error Handling**:
```rust
#[derive(thiserror::Error, Debug)]
pub enum ParseError {
    #[error("Invalid date at line {line}: {date}")]
    InvalidDate { line: usize, date: String },
    
    #[error("Unbalanced transaction at line {line}: {diff}")]
    UnbalancedTransaction { line: usize, diff: Decimal },
    
    // ... comprehensive error types with context
}
```

---

### `rledger-storage`
**Purpose**: Data persistence and caching  
**Exports**: Storage engine, migrations, queries  
**Dependencies**: rledger-common, rledger-core, sqlx, tokio

**Schema Design** (SQLite):
```sql
-- Accounts table
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    account_type TEXT NOT NULL,
    parent_id INTEGER REFERENCES accounts(id),
    commodity TEXT,
    created_at TEXT NOT NULL
);
CREATE INDEX idx_accounts_name ON accounts(name);

-- Transactions table
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY,
    date TEXT NOT NULL,  -- ISO 8601
    status TEXT NOT NULL,
    code TEXT,
    description TEXT NOT NULL,
    source_file TEXT,
    source_line INTEGER,
    created_at TEXT NOT NULL
);
CREATE INDEX idx_transactions_date ON transactions(date);

-- Postings table
CREATE TABLE postings (
    id INTEGER PRIMARY KEY,
    transaction_id INTEGER NOT NULL REFERENCES transactions(id),
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    amount_quantity TEXT,  -- Decimal as string for precision
    amount_commodity TEXT,
    cost_quantity TEXT,
    cost_commodity TEXT,
    posting_order INTEGER NOT NULL
);
CREATE INDEX idx_postings_transaction ON postings(transaction_id);
CREATE INDEX idx_postings_account ON postings(account_id);

-- Balance cache
CREATE TABLE balance_cache (
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    date TEXT NOT NULL,
    commodity TEXT NOT NULL,
    balance TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    PRIMARY KEY (account_id, date, commodity)
);
```

**Performance Optimizations**:
- WAL mode: `PRAGMA journal_mode=WAL`
- Batch inserts: 1000 transactions per batch
- Prepared statements cached
- Strategic indexing on date, account_id
- Balance cache with invalidation

**Cache Strategy**:
```rust
// Cache hierarchy:
// 1. In-memory HashMap (hot data)
// 2. SQLite balance_cache (warm data)
// 3. Recompute from transactions (cold)

pub struct BalanceCache {
    // LRU cache in memory
    memory: HashMap<CacheKey, Balance>,
    // SQLite for persistence
    db: Pool<Sqlite>,
}

// Invalidation: When new transaction added, invalidate affected accounts
```

---

### `rledger-cli`
**Purpose**: Command-line interface  
**Exports**: Binary (`rledger`)  
**Dependencies**: All other crates, clap, colored, tokio

**Commands**:
```rust
#[derive(Subcommand)]
enum Commands {
    /// Show account balances
    Balance {
        accounts: Vec<String>,
        #[arg(long)] depth: Option<usize>,
        #[arg(long)] flat: bool,
    },
    
    /// Show transaction register
    Register {
        account: String,
        #[arg(short = 'H', long)] historical: bool,
    },
    
    /// Print transactions in canonical format
    Print {
        #[arg(long)] description: Option<String>,
    },
    
    /// List accounts
    Accounts {
        pattern: Option<String>,
    },
    
    /// Show journal statistics
    Stats,
}
```

**Output Formatting**:
- Colored terminal output (via `colored` crate)
- Tree view for balances (hierarchical accounts)
- CSV export (`-O csv`)
- JSON export (`-O json`)
- Pretty-printed amounts with commodity symbols

---

## Data Models

### Core Types

#### AccountName
```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct AccountName(String);

impl AccountName {
    pub fn new(name: impl Into<String>) -> Self;
    pub fn as_str(&self) -> &str;
    pub fn components(&self) -> Vec<&str>;  // Split by ':'
    pub fn parent(&self) -> Option<AccountName>;
    pub fn is_child_of(&self, other: &AccountName) -> bool;
}
```

**Design**: Newtype for type safety, string interning for performance (future optimization).

#### Amount
```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Amount {
    pub quantity: Decimal,
    pub commodity: Commodity,
}

impl Amount {
    pub fn new(quantity: Decimal, commodity: impl Into<Commodity>) -> Self;
    pub fn is_zero(&self) -> bool;
    pub fn abs(&self) -> Self;
    pub fn negate(&self) -> Self;
}
```

**Critical**: Always use `Decimal`, never `f64`. Example: `0.1 + 0.2 = 0.30000000000000004` in f64, but `0.3` exactly in Decimal.

#### Transaction
```rust
#[derive(Debug, Clone)]
pub struct Transaction {
    pub date: NaiveDate,
    pub status: Status,
    pub code: Option<String>,  // Optional transaction code (123)
    pub description: String,
    pub postings: Vec<Posting>,
    pub metadata: HashMap<String, String>,  // Key-value tags
}

impl Transaction {
    pub fn is_balanced(&self) -> Result<(), ValidationError>;
    pub fn auto_balance(&mut self) -> Result<(), ValidationError>;
    pub fn affected_accounts(&self) -> Vec<&AccountName>;
}
```

---

## Coding Standards

### Rust Style

**Formatting**:
- Use `rustfmt` with default settings
- Run `cargo fmt --all` before commits
- CI enforces formatting

**Linting**:
- Zero `clippy` warnings: `cargo clippy --all -- -D warnings`
- Allow specific clippy lints only with justification comment

**Naming**:
- Types: `PascalCase`
- Functions/methods: `snake_case`
- Constants: `SCREAMING_SNAKE_CASE`
- Modules: `snake_case`

### Error Handling

**Libraries (`rledger-core`, `rledger-storage`)**: Use `thiserror`
```rust
#[derive(thiserror::Error, Debug)]
pub enum ParseError {
    #[error("Invalid date at line {line}: {date}")]
    InvalidDate { line: usize, date: String },
}
```

**Application (`rledger-cli`)**: Use `anyhow`
```rust
fn main() -> anyhow::Result<()> {
    let journal = load_journal(&path)
        .context("Failed to load journal")?;
    Ok(())
}
```

**Never**:
- Don't use `unwrap()` or `expect()` in production paths
- Don't use `panic!()` for recoverable errors
- Don't silently ignore errors

### Documentation

**Every public item needs docs**:
```rust
/// Calculate the balance for an account as of a specific date.
///
/// # Arguments
/// * `account` - The account name
/// * `as_of` - The date to calculate balance
///
/// # Returns
/// A HashMap mapping commodities to their balances
///
/// # Examples
/// ```
/// use rledger_core::*;
/// let balance = calculate_balance(&account, date)?;
/// ```
///
/// # Errors
/// Returns `CalculationError` if account doesn't exist
pub fn calculate_balance(
    account: &AccountName,
    as_of: NaiveDate,
) -> Result<HashMap<Commodity, Decimal>, CalculationError> {
    // implementation
}
```

### Testing

**Unit Tests**: In same file as code
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_parse_date_valid() {
        let result = parse_date("2024-01-15");
        assert!(result.is_ok());
    }
    
    #[test]
    fn test_transaction_balances() {
        // Arrange
        let mut txn = Transaction::new(date);
        txn.add_posting(/* ... */);
        
        // Act
        let result = txn.is_balanced();
        
        // Assert
        assert!(result.is_ok());
    }
}
```

**Property-Based Tests**: For algorithms
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_parse_date_roundtrip(
        year in 2000i32..2100,
        month in 1u32..=12,
        day in 1u32..=28
    ) {
        let date = NaiveDate::from_ymd_opt(year, month, day).unwrap();
        let formatted = format_date(&date);
        let parsed = parse_date(&formatted).unwrap();
        prop_assert_eq!(date, parsed);
    }
}
```

**Integration Tests**: In `tests/` directory
```rust
// tests/integration/basic_workflow.rs
#[test]
fn test_full_workflow() {
    // 1. Create test journal
    // 2. Parse
    // 3. Validate
    // 4. Generate reports
    // 5. Assert expected output
}
```

---

## Decision Log

### ADR-001: Use Rust for Implementation
**Date**: 2025-09  
**Status**: Accepted  
**Context**: Need performance, memory safety, modern tooling  
**Decision**: Implement in Rust instead of C++, Go, or Haskell  
**Consequences**: 
- ✅ Memory safety without GC
- ✅ Best-in-class performance
- ✅ Modern package management (Cargo)
- ❌ Steeper learning curve for contributors

### ADR-002: SQLite for Caching, Not Primary Storage
**Date**: 2025-09  
**Status**: Accepted  
**Context**: Need to balance performance with plain-text philosophy  
**Decision**: Plain-text journal is source of truth, SQLite is cache  
**Consequences**:
- ✅ Users maintain control of data
- ✅ Git-friendly diffs
- ✅ Can regenerate cache if corrupted
- ✅ Performance benefits from indexing
- ⚠️ Must handle cache invalidation correctly

### ADR-003: nom for Parsing
**Date**: 2025-09  
**Status**: Accepted  
**Context**: Need fast, maintainable parser  
**Decision**: Use nom parser combinators  
**Alternatives Considered**: pest (PEG), hand-written recursive descent  
**Consequences**:
- ✅ Zero-copy parsing
- ✅ Excellent error messages
- ✅ Composable parsers
- ❌ Learning curve for nom

### ADR-004: rust_decimal for Amounts
**Date**: 2025-09  
**Status**: Accepted  
**Context**: Need exact decimal arithmetic  
**Decision**: Use rust_decimal crate  
**Consequences**:
- ✅ No floating-point errors
- ✅ Financial-grade precision
- ✅ Well-tested library
- ⚠️ Slightly slower than f64 (acceptable trade-off)

### ADR-005: Workspace with Separate Crates
**Date**: 2025-09  
**Status**: Accepted  
**Context**: Need modular, maintainable codebase  
**Decision**: Use Cargo workspace with 4 crates  
**Consequences**:
- ✅ Clear separation of concerns
- ✅ Testable in isolation
- ✅ Reusable components
- ⚠️ More boilerplate initially

### ADR-006: Rust Edition 2024
**Date**: 2025-09  
**Status**: Accepted  
**Context**: New project, want latest features  
**Decision**: Use Rust 2024 edition, MSRV 1.90  
**Consequences**:
- ✅ Latest language features (let chains, etc.)
- ✅ Best tooling support
- ⚠️ Requires recent Rust (acceptable for new project)

---

## Implementation Guidelines

### Phase 1 Priorities (Current)

**Week 1-2**: Core data structures and parser
- [ ] Define all types in `rledger-common`
- [ ] Implement nom-based parser in `rledger-core`
- [ ] 100+ unit tests for parser
- [ ] Handle 80% of ledger/hledger syntax

**Week 3-4**: Validation and accounting logic
- [ ] Transaction balancing algorithm
- [ ] Account hierarchy management
- [ ] Balance assertion checking
- [ ] Auto-balance single omitted posting

**Week 5-6**: Storage layer
- [ ] SQLite schema and migrations
- [ ] Batch insert optimizations
- [ ] Balance caching with invalidation
- [ ] Integration tests

**Week 7-8**: CLI implementation
- [ ] clap-based command structure
- [ ] `balance` command with tree/flat views
- [ ] `register` command
- [ ] `print` command
- [ ] Colored output

**Week 9-10**: Testing and compatibility
- [ ] Test with real ledger/hledger files
- [ ] Ensure output parity
- [ ] Performance benchmarks
- [ ] Bug fixes

**Week 11-12**: Documentation and release
- [ ] User documentation
- [ ] API documentation
- [ ] Migration guides
- [ ] v0.1.0-alpha.1 release

### File Format Compatibility

**Must Support** (Phase 1):
```ledger
; Comments
2024-01-01 * Opening Balances
    Assets:Checking          $1,000.00
    Equity:Opening

2024-01-15 ! (123) Pending Transaction
    Assets:Checking          $500.00
    Income:Salary           ; auto-balanced

; Include other files
include 2024/expenses.journal

; Price directives
P 2024-01-01 AAPL $150.00

; Account directives
account Assets:Investments
    note: Brokerage account

; Commodity directives
commodity $
    format $1,000.00
```

**Not Required** (Phase 1):
- Automated transactions
- Periodic transactions
- Budget directives
- Value expressions
- Python expressions

### Performance Targets

| Operation | Target | Measurement |
|-----------|--------|-------------|
| Parse 10k txns | <200ms | `cargo bench --bench parsing` |
| Parse 100k txns | <2s | `cargo bench --bench parsing` |
| Balance query | <100ms | For 10k txns |
| Register query | <50ms | Single account |
| Report generation | <500ms | Balance sheet, 10k txns |
| Startup time | <100ms | Cold start |
| Memory usage | <100MB | 100k txns in memory |

### Testing Strategy

**Coverage Target**: 80%+ overall, 100% for financial calculations

**Test Pyramid**:
- 70% Unit tests (fast, isolated)
- 20% Integration tests (end-to-end)
- 10% Compatibility tests (vs ledger/hledger)

**Continuous Integration**:
- Run on every PR
- Test on Linux, macOS, Windows
- Clippy with warnings as errors
- rustfmt check
- Security audit (cargo-audit)

---

## Quick Reference

### Common Commands
```bash
# Development
cargo check --all               # Fast compilation check
cargo test --all                # Run all tests
cargo clippy --all -- -D warnings  # Lint
cargo fmt --all                 # Format code

# Performance
cargo bench                     # Run benchmarks
cargo flamegraph --bench parsing # Profile

# Release
cargo build --release --all     # Optimized build
```

### File Locations
```
rledger/
├── Cargo.toml                  # Workspace config
├── crates/
│   ├── rledger-common/         # Shared types
│   ├── rledger-core/           # Parser & validation
│   ├── rledger-storage/        # SQLite
│   └── rledger-cli/            # CLI app
├── tests/integration/          # E2E tests
├── benches/                    # Benchmarks
└── docs/                       # Documentation
```

### Key Files to Reference
- **Architecture**: `docs/architecture.md`
- **Parser Spec**: `docs/parser-spec.md`
- **API Docs**: Run `cargo doc --open`
- **Examples**: `examples/` directory

---

## For AI Assistants

When generating code for rledger, always:

1. **Check this document first** for architectural decisions
2. **Use workspace dependencies** (`{ workspace = true }`)
3. **Follow error handling patterns** (thiserror for libs, anyhow for apps)
4. **Add comprehensive tests** for all new code
5. **Document public APIs** with examples
6. **Use `Decimal` for money**, never `f64`
7. **Prefer immutable data structures**
8. **Add tracing** for debugging: `tracing::debug!("message")`
9. **Consider performance** but prioritize correctness first
10. **Ask for clarification** if requirements conflict

**Remember**: This is a financial application. Correctness > Performance > Convenience.

---

**End of Technical Specification**

*This document is the source of truth for architectural and PM decisions. Update it as decisions are made.*
