# Click Reference Documentation

Personal reference documentation for Click - the multi-channel trading platform for Solana and Hyperliquid.

## Contents

| Document | Description |
|----------|-------------|
| [01-what-is-click.md](01-what-is-click.md) | Company overview, mission, target users |
| [02-products.md](02-products.md) | All products: Web Terminal, Telegram Bots, Extension, Wallet |
| [03-architecture.md](03-architecture.md) | Technical architecture and infrastructure |
| [04-web-terminal-features.md](04-web-terminal-features.md) | Web Terminal feature deep dive |
| [05-data-and-metrics.md](05-data-and-metrics.md) | All data fields and metrics across products |
| [06-clickhouse-requirements.md](06-clickhouse-requirements.md) | ClickHouse storage and query requirements |
| [07-competitive-analysis.md](07-competitive-analysis.md) | Competitive positioning and differentiators |

## Quick Reference

**What**: Multi-channel, non-custodial trading platform for Solana meme coins & Hyperliquid perps

**Who**: Solana traders ("degens") who need speed, information density, and privacy

**Products**:
- Web Terminal (primary)
- Telegram Bot (spot trading)
- Telegram Perps Bot (Hyperliquid)
- Chrome Extension (overlay for other terminals)
- Click Wallet (non-custodial, shared across all)
- Multi-Wallet (up to 100 wallets + privacy transfers)

**Tech Stack**:
- Bare Metal: Solana Engine, DataStreams, ClickHouse, NATS, Redis
- AWS: Signer (Nitro Enclave), REST API, Postgres
- Products: React (Web), Node.js (Telegram), Chrome Extension

**Tagline**: "Trade like a whale without looking like one"

---

*Generated from Click's internal documentation repository*
