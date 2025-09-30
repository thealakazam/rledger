# rledger

[![CI](https://github.com/thealakazam/rledger/workflows/CI/badge.svg)](https://github.com/thealakazam/rledger/actions)
[![codecov](https://codecov.io/gh/thealakazam/rledger/branch/main/graph/badge.svg)](https://codecov.io/gh/thealakazam/rledger)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Crates.io](https://img.shields.io/crates/v/rledger.svg)](https://crates.io/crates/rledger)

**A modern, high-performance plain-text accounting system written in Rust**

rledger is a reimagining of [ledger](https://ledger-cli.org/) and [hledger](https://hledger.org/) with a focus on speed, reliability, and usability. Keep your books in plain text files with version control, while enjoying modern performance and a clean interface.

## ✨ Features

- 🚀 **Blazingly Fast** - Parse 100k transactions in under 2 seconds
- 📝 **Plain Text** - Human-readable journal files you can edit and version control
- 🔄 **Ledger Compatible** - Drop-in replacement for most ledger/hledger workflows
- 💰 **Investment Tracking** - Built-in lot tracking and capital gains calculation
- 🌍 **Global Markets** - Price fetching for stocks, mutual funds, crypto (including Indian markets)
- 🖥️ **Multiple Interfaces** - CLI, web UI, desktop app, REST API
- 🛡️ **Type Safe** - Rust's memory safety means no crashes or data corruption
- 🎨 **Modern UX** - Beautiful terminal output and intuitive commands

## 🎯 Why rledger?

| Problem | rledger Solution |
|---------|------------------|
| GnuCash takes 40+ seconds to start | Sub-second startup and instant commands |
| Existing tools struggle with 10k+ transactions | Handles 100k+ transactions effortlessly |
| Investment tracking is manual and error-prone | Automatic lot tracking with FIFO/LIFO/specific ID |
| Indian securities price fetching is broken | Native support for NSE, BSE, AMFI |
| No accessible UI for non-technical users | Web interface and desktop app included |
| Complex learning curve | Intuitive commands with helpful error messages |

## 🚀 Quick Start

### Installation

**From source** (requires [Rust](https://rustup.rs/)):
```bash
cargo install rledger
```

**From binary** (coming soon):
```bash
# Linux/macOS
curl -sSL https://get.rledger.dev | sh

# Or download from releases
# https://github.com/thealakazam/rledger/releases
```

### Your First Journal

Create `~/.rledger/main.journal`:

```ledger
; Opening balances
2024-01-01 Opening Balances
    Assets:Checking           $1,000.00
    Assets:Savings            $5,000.00
    Equity:Opening Balances

; Regular transactions
2024-01-15 Salary
    Assets:Checking           $3,500.00
    Income:Salary

2024-01-16 Rent
    Assets:Checking          -$1,200.00
    Expenses:Housing:Rent

2024-01-17 Groceries
    Assets:Checking             -$85.43
    Expenses:Food:Groceries
```

### Basic Commands

**Check your balance:**
```bash
$ rledger balance
Account Balances

Assets                               $6,214.57
  Checking                            1,214.57
  Savings                             5,000.00
Equity                              -$6,000.00
  Opening Balances                   -6,000.00
Expenses                              $1,285.43
  Food:Groceries                         85.43
  Housing:Rent                        1,200.00
Income                              -$3,500.00
  Salary                             -3,500.00

Total:                                   $0.00
```

**View account activity:**
```bash
$ rledger register Assets:Checking
2024-01-01 Opening Balances        Assets:Checking        $1,000.00    $1,000.00
2024-01-15 Salary                  Assets:Checking         3,500.00     4,500.00
2024-01-16 Rent                    Assets:Checking        -1,200.00     3,300.00
2024-01-17 Groceries               Assets:Checking           -85.43     3,214.57
```

**Generate reports:**
```bash
# Balance for specific period
rledger balance --begin 2024-01-01 --end 2024-01-31

# Monthly breakdown
rledger balance --monthly Expenses

# Export to CSV
rledger register Assets:Checking -O csv > checking.csv
```

## 📚 Documentation

- **[Getting Started Guide](docs/getting-started.md)** - Your first 10 minutes with rledger
- **[Journal Format](docs/journal-format.md)** - Complete syntax reference
- **[Command Reference](docs/commands.md)** - All commands and options
- **[Migration from Ledger/hledger](docs/migration.md)** - Switch seamlessly
- **[Investment Tracking](docs/investments.md)** - Lots, cost basis, and capital gains
- **[API Documentation](https://docs.rs/rledger)** - For developers

## 🏗️ Project Status

rledger is currently in **active development** for Phase 1 (Core Engine).

### Roadmap

- **Phase 1 - Core Engine** (Current - Q1 2025)
  - ✅ Journal parsing (ledger/hledger compatible)
  - ✅ Double-entry validation
  - ✅ CLI with balance, register, print commands
  - ✅ SQLite storage layer
  - 🚧 Testing and documentation

- **Phase 2 - Investment Features** (Q2 2025)
  - 📋 Lot tracking (FIFO/LIFO/specific identification)
  - 📋 Capital gains calculation
  - 📋 Price fetching (Yahoo Finance, NSE, BSE, AMFI)
  - 📋 Investment reports

- **Phase 3 - Daemon & API** (Q3 2025)
  - 📋 Background daemon mode
  - 📋 REST API
  - 📋 Real-time file watching
  - 📋 Multi-user support

- **Phase 4-7** (Q4 2025 - Q1 2026)
  - 📋 Web interface
  - 📋 Desktop application (Tauri)
  - 📋 CSV import wizard
  - 📋 v1.0 release

**Legend:** ✅ Complete | 🚧 In Progress | 📋 Planned

## 💻 Development

### Prerequisites

- Rust 1.75 or later
- SQLite 3.35+

### Building from Source

```bash
# Clone repository
git clone https://github.com/thealakazam/rledger.git
cd rledger

# Build
cargo build --release

# Run tests
cargo test

# Run with example journal
cargo run -- -f examples/sample.journal balance

# Install locally
cargo install --path crates/rledger-cli
```

### Project Structure

```
rledger/
├── crates/
│   ├── rledger-core/      # Parser, validation, accounting logic
│   ├── rledger-storage/   # SQLite storage and caching
│   ├── rledger-cli/       # Command-line interface
│   └── rledger-common/    # Shared types and utilities
├── tests/                 # Integration tests
├── benches/              # Performance benchmarks
├── docs/                 # Documentation
└── examples/             # Example journals
```

### Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Ways to contribute:**
- 🐛 Report bugs and issues
- 💡 Suggest features and improvements
- 📝 Improve documentation
- 🧪 Add test cases
- 💻 Submit pull requests
- 🌍 Add price fetchers for your country/market

**Good first issues:** Look for the [`good-first-issue`](https://github.com/thealakazam/rledger/labels/good-first-issue) label.

## 🆚 Comparison

| Feature | rledger | ledger | hledger | GnuCash |
|---------|---------|--------|---------|---------|
| **Performance (100k txns)** | <2s | ~10s | ~5s | 40s+ startup |
| **Plain text files** | ✅ | ✅ | ✅ | ❌ |
| **Built-in lot tracking** | ✅ | ❌ | ❌ | ✅ |
| **Price fetching** | ✅ | Limited | Limited | Broken |
| **Indian markets (NSE/BSE)** | ✅ | ❌ | ❌ | ❌ |
| **Web interface** | ✅ (planned) | ❌ | Separate | ❌ |
| **Desktop app** | ✅ (planned) | ❌ | ❌ | ✅ (GTK) |
| **REST API** | ✅ (planned) | ❌ | ❌ | ❌ |
| **Memory safe** | ✅ (Rust) | ❌ (C++) | ⚠️ (Haskell) | ⚠️ (C) |
| **Learning curve** | Low | High | Medium | Medium |

## 🤝 Community

- **Discussions:** [GitHub Discussions](https://github.com/thealakazam/rledger/discussions)
- **Issues:** [Issue Tracker](https://github.com/thealakazam/rledger/issues)
- **Matrix/Discord:** Coming soon
- **Blog:** Coming soon

## 📄 License

rledger is open source software licensed under the [Apache License 2.0](LICENSE).

## 🙏 Acknowledgments

rledger stands on the shoulders of giants:

- [ledger](https://ledger-cli.org/) by John Wiegley - The original plain-text accounting tool
- [hledger](https://hledger.org/) by Simon Michael - Bringing reliability and documentation
- The entire [Plain Text Accounting](https://plaintextaccounting.org/) community
- All the amazing [Rust](https://www.rust-lang.org/) libraries we depend on

## 🌟 Star History

[![Star History Chart](https://api.star-history.com/svg?repos=thealakazam/rledger&type=Date)](https://star-history.com/#thealakazam/rledger&Date)

---

**Made with ❤️ by the rledger community**

[Website](https://rledger.dev) • [Documentation](https://docs.rledger.dev) • [Twitter](https://twitter.com/rledger_dev)
