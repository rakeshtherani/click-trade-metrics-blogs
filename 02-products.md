# Click Products

## Product Portfolio Overview

```
USER-FACING PRODUCTS (B2C)
├── Web Terminal (Primary Product)
│   ├── Discovery Page
│   ├── Token Trading Page
│   ├── Portfolio Page
│   ├── Trending Page
│   ├── Settings Page
│   └── Perps Module
├── Telegram Bot (Spot Trading)
├── Telegram Perps Bot (Hyperliquid)
├── Chrome Extension
├── Click Wallet
└── Multi-Wallet Management

INTERNAL SERVICES
├── REST API (Gateway)
├── Solana Engine (Indexer + Execution)
├── Hyperliquid Indexer
├── DataStreams (Real-time Metrics)
├── Signer (Nitro Enclave)
├── ClickHouse (Analytics DB)
├── NATS (Message Bus)
├── Redis (Cache)
└── Postgres (User Data)
```

---

## 1. Web Terminal (Primary Product)

**Purpose**: Full-featured trading dashboard for desktop power users

**URL**: click.trade (Vercel)

### 1.1 Discovery Page

**Purpose**: Find new tokens across ALL Solana launchpads

**Three Lifecycle Sections**:

| Section | Definition | Sorting |
|---------|------------|---------|
| **Newly Created** | Just deployed, bonding <60% | Newest first |
| **About to Graduate** | $40K-$60K mcap, bonding 60-100% | By graduation proximity |
| **Graduated** | Completed bonding, on DEX pools | By volume |

**Token Card Metrics**:
- Token image (with bonding curve % ring)
- Name, ticker, contract address (copyable)
- Deployment age (e.g., "2m ago")
- Market cap
- 24h volume
- Holder count
- Insider count (wallets without buy tx)
- Pro trader count (bots detected)
- Social links (hover for iframe preview)
- Quick Buy button (one-click trade)

**Advanced Metrics (Tooltips)**:
- Dev holdings % (or "Dev Sold")
- Sniper % (bought same slot as creation)
- Top 10 holder %
- Bundler % (coordinated wallets)
- Insider % (received via transfer)
- Rug risk score

**Supported Launchpads (9+)**:
Pump.fun, Bonk, MoonIt, Believe, Jupiter, MoonE, Boop, Launchlabs, Meteora (Dynamic)

**Filters**:
- Launchpad selection
- Keywords include/exclude
- Bonding curve % range
- Age range
- Top 10 % range
- Dev holdings % range
- Liquidity range
- Volume range
- Market cap range
- Social requirements (Twitter/Website/Telegram)

### 1.2 Token Trading Page

**Purpose**: Full trading interface for a single token

**Entry Points**: Discovery page, search bar, watchlist, direct link

**Chart Features**:
- 14 timeframes: 1s, 5s, 15s, 30s, 1m, 5m, 15m, 30m, 1h, 4h, 6h, 12h, 1d, 1w
- Load time target: <0.5 seconds
- Overlays: Average entry, limit orders, TP/SL levels
- Markers: Your buys/sells, tracked wallet activity
- Events: Graduation, LP additions

**Token Info Panel**:
| Field | Description |
|-------|-------------|
| Top 10 holder % | Concentration |
| Dev holdings % | Dev position |
| Sniper/Insider/Bundler % | Risk metrics |
| Phishing % | Known bad actors |
| Dex Paid | DexScreener verification |
| No Mint | Mint authority disabled |
| Burned LP % | Rug protection signal |
| Rug risk % | Multi-factor score |

**Trading Panel**:
- PnL module: Bought, Sold, Holding, Current PnL
- Order types: Market, Limit, Take-Profit, Stop-Loss
- Multi-wallet selector (spread across wallets)
- Configurable presets (3 configurations)
- Slippage, fees, MEV protection per order

**Tabs**:
| Tab | Content |
|-----|---------|
| Trades | Real-time transaction feed |
| Holders | Top 30 with classifications & PnL |
| Positions | Your positions across all tokens |
| Orders | Active limit orders |
| History | All your buy/sell transactions |
| Dev Tokens | Other tokens by same developer |

**Wallet Classification Badges**:
- Sniper: Bought in creation slot
- Insider: Received via transfer (no buy)
- Bundler: Coordinated wallet group
- Fresh Wallet: <7 days old, <10 txs
- Dev: Deployed token or initial supply
- KOL: Key Opinion Leader
- Top 10: In top 10 by balance

### 1.3 Portfolio Page

**Purpose**: Track positions and performance

**Spot Portfolio**:
- Total Value (SOL + USD)
- Available Balance
- Unrealized PnL
- Realized PnL
- Timeframe filter: 1D, 7D, 30D, Max

**Performance Categories (Realized PnL)**:
| Category | Definition |
|----------|------------|
| Moonshot | >5x gain (>500%) |
| Pump | 2-5x gain (100-500%) |
| Win | 1-2x gain (0-100%) |
| Loss | 0% to -50% |
| Rekt | >50% loss |

**Position Tabs**:
- Active: Token, Bought, Sold, Remaining, PnL, Actions
- Closed: Completed positions
- Hidden: Archived (still in totals)
- Activity: All trade history

**Perps Portfolio**:
- All-Time Volume, PnL, Trade Count, Account Value
- Open Positions: Coin, Direction, Size, Entry, Mark, PnL, Liquidation
- Trade History

**PnL Sharing**:
- Generate shareable cards
- Copy/Download/Tweet

### 1.4 Trending Page

**Purpose**: Market signals and whale tracking

**Features**:
- Volume trending tokens
- Whale activity feed
- Social signals (calls)
- Market sentiment

### 1.5 Settings Page

**Purpose**: Configure all preferences

**Settings**:
- Slippage defaults
- Priority fees (buy/sell)
- MEV protection
- UI presets (buy/sell amounts)
- Notifications
- Multi-wallet management

---

## 2. Telegram Bot (Spot Trading)

**Purpose**: Non-custodial Solana trading directly in Telegram

**Bot**: @clickdottrade_bot

### Key Features

| Feature | Description |
|---------|-------------|
| Wallet Creation | Click wallet, seed in secure mini app |
| Trading Intent | One-time approval for fast execution |
| Market Orders | Buy/sell with presets or custom |
| Limit Orders | Trigger on price/mcap/multiple/% |
| DCA Orders | Automated recurring buys |
| Copy Trading | Follow wallets with filters |
| Alerts | Price/mcap notifications |
| Points & Referrals | Earn rewards |
| PnL Cards | Shareable wins |

### Commands

| Command | Function |
|---------|----------|
| `/start`, `/home` | Dashboard |
| `/buy <amount> <CA>` | Buy order |
| `/sell <percent> <CA>` | Sell order |
| `/positions` | View holdings |
| `/wallet` | Wallet info |
| `/limit` | Limit orders |
| `/dca` | DCA orders |
| `/copytrade` | Copy strategies |
| `/alerts` | Price alerts |
| `/points` | Earned points |
| `/referrals` | Referral rewards |
| `/settings` | Preferences |
| `/autobuy` | Toggle auto-buy |

### Token Trading Panel (on CA paste)
- Token name, ticker, DEX source
- Market cap (current + ATH)
- Price (SOL + USD)
- Liquidity, 24h volume
- 5m/1h/6h/24h price changes
- Your limit orders
- Tabs: DCA, Market, Limit

### Copy Trading Filters
- Buy Once flag
- Require renounced authorities
- Min/max market cap
- Min liquidity
- Max buy per token
- Copy sells toggle

---

## 3. Telegram Perps Bot

**Purpose**: Hyperliquid perpetual futures via Telegram

**Bot**: @clickdottrade_perps_bot

### Key Features

| Feature | Description |
|---------|-------------|
| Click EVM Wallet | Same wallet as Web Terminal |
| Slash Commands | /long, /short, /close, /sl, /tp |
| Order Types | Market, Limit with leverage |
| Margin Modes | Isolated or Cross |
| PnL Cards | Position and portfolio cards |
| Group Broadcasting (V2) | Post trades to groups |
| Copy Trading (V3) | By wallet or username |

### Commands

| Command | Function |
|---------|----------|
| `/long [ticker]` | Open long |
| `/short [ticker]` | Open short |
| `/positions` | View positions |
| `/orders` | Pending orders |
| `/close` | Close position |
| `/sl` | Set stop-loss |
| `/tp` | Set take-profit |
| `/portfolio` | Portfolio view |
| `/wallet` | Wallet info |
| `/deposit` | USDC deposit address |
| `/withdraw` | Withdraw to EVM |
| `/settings` | Broadcasting, copy opt-in |

### Copy Trading Modes
- **By Wallet Address**: No opt-in, no profit share
- **By Username**: Leader opts in, 10% profit share
- Each copy creates isolated sub-wallet

### Version Roadmap
- V1: Core perps trading
- V2: Group broadcasting
- V3: Copy trading

---

## 4. Chrome Extension

**Purpose**: Trading overlay for third-party terminals

**Core Value**: Keep your favorite discovery terminal, use Click's execution

### Supported Sites
- Axiom, GMGN, Photon, Dexscreener
- Discord web, Solscan, Pump.fun
- Telegram web, Twitter/X
- Padre, Telemetry

### Key Features

| Feature | Description |
|---------|-------------|
| Quick Buy Injection | Buttons on supported sites |
| Mini Trade Terminal | Draggable overlay |
| Positions Panel | Live positions with PnL |
| Site Detection | Auto-detect token from page |
| Per-Site Toggle | Enable/disable per domain |
| Position Persistence | Remembers overlay positions |

### Extension Popup
- Master toggle (global on/off)
- Preset configuration (4 buy, 4 sell)
- Settings: slippage, fees, tip
- Audio alerts toggle
- Logout

### Requirements
- Telegram authentication required
- Desktop Chrome only (V1)
- Dark mode only (V1)

---

## 5. Click Wallet

**Purpose**: Non-custodial wallet powering all products

### Key Features

| Feature | Description |
|---------|-------------|
| Unified Wallet | Same wallet across all products |
| Non-Custodial | You own your keys |
| Intent-Based Signing | Approve once, trade fast |
| Seed in Mini App | Never shown in chat |
| Multi-Wallet | Up to 100 wallets |

### Signer Service (Nitro Enclave)
- TEE-based transaction signing
- Keys never stored in raw form
- Intent tokens for fast execution

---

## 6. Multi-Wallet Management

**Purpose**: Whale-grade privacy and operational security

**Core Value**: "Trade like a whale without looking like one"

### Key Features

| Feature | Description |
|---------|-------------|
| Create Wallet | Up to 100 per account |
| Import Wallet | Via private key |
| Normal Transfer | Standard Solana transfer |
| Private Transfer | Houdini swap (unlinkable) |
| Spread Trading | Distribute across wallets |
| Unified Portfolio | Aggregated view |

### Private Transfer (Houdini Swap)
- 1% total fee (0.8% to Click, 0.2% to Houdini)
- Breaks on-chain linkage
- 30-60 second latency

### Spread Trading
- Multi-select wallets
- Randomized distribution
- Individual orders per wallet
- Per-wallet status tracking

---

## Product Comparison

| Feature | Web Terminal | TG Bot | TG Perps | Extension |
|---------|--------------|--------|----------|-----------|
| Token Discovery | ✅ Full | ❌ | ❌ | ❌ (uses host) |
| Charts | ✅ 14 TF | ❌ | ❌ | ❌ |
| Market Orders | ✅ | ✅ | ✅ | ✅ |
| Limit Orders | ✅ | ✅ | ✅ | ❌ V1 |
| DCA | ✅ | ✅ | ❌ | ❌ |
| Copy Trading | ✅ | ✅ | ✅ V3 | ❌ |
| Portfolio | ✅ | ✅ | ✅ | ✅ (mini) |
| Perps | ✅ | ❌ | ✅ | ❌ |
| Multi-Wallet | ✅ | V2 | ❌ | ❌ V1 |
| Holder Analytics | ✅ | ❌ | ❌ | ❌ |

---

## Internal Services

| Service | Purpose |
|---------|---------|
| **REST API** | Gateway for all products |
| **Solana Engine** | Blockchain indexing + order execution |
| **Hyperliquid Indexer** | Perps data indexing |
| **DataStreams** | Real-time metric computation |
| **Signer** | Non-custodial signing (Nitro Enclave) |
| **ClickHouse** | Historical data storage |
| **NATS** | Message bus |
| **Redis** | Cache, sessions |
| **Postgres** | User data, orders |
