# Web Terminal Features Deep Dive

## Overview

The Web Terminal is Click's primary product - a full-featured trading dashboard for desktop power users.

---

## 1. Discovery Page

### Purpose
Surface the most relevant Solana meme tokens at every lifecycle stage with all critical metrics visible at a glance.

### Three Lifecycle Sections

| Section | Definition | Market Cap | Bonding Curve | Sorting |
|---------|------------|------------|---------------|---------|
| **Newly Created** | Just deployed | Any | <60% | Newest first |
| **About to Graduate** | Near graduation | $40K-$60K | 60-100% | By graduation proximity |
| **Graduated** | On DEX pools | Any | 100% | By volume |

### Token Card - Visible Metrics

| Metric | Description |
|--------|-------------|
| Token Image | With bonding curve % ring indicator |
| Token Name | Full name |
| Token Ticker | Symbol |
| Contract Address | Copyable |
| Deployment Age | e.g., "2m ago" |
| Market Cap | Current USD value |
| 24h Volume | Trading volume |
| Holder Count | Unique wallets with balance |
| Insider Count | Wallets without buy tx |
| Pro Trader Count | Bots detected |
| Global Fees Paid | Total swap fees |
| Social Links | Website, Twitter, Telegram, TikTok |

### Token Card - Advanced Metrics (Tooltips)

| Metric | Description |
|--------|-------------|
| Dev Holdings % | Or "Dev Sold" indicator |
| Sniper % | Supply held by same-slot buyers |
| Top 10 Holder % | Concentration in top 10 |
| Bundler % | Coordinated wallet groups |
| Insider % | Supply without buy tx |

### Social Link Previews
- Hover over any social link shows iframe preview
- Supported: Twitter/X, Telegram, Website, TikTok, pump.fun livestream
- No tab switching needed

### Quick Buy
- Each section has configurable preset amount (in SOL)
- One-click execution directly from card
- Amounts persist per section across sessions
- Transaction feedback: sending → confirmed → failed

### Filter System

| Filter | Options |
|--------|---------|
| Launchpad | Pump, Bonk, MoonIt, Believe, Jupiter, MoonE, Boop, Launchlabs, Meteora |
| Keywords Include | Text search |
| Keywords Exclude | Exclude terms |
| Bonding Curve % | Min / Max |
| Age | Min / Max |
| Top 10 % | Min / Max |
| Dev Holdings % | Min / Max |
| Liquidity | Min / Max |
| Volume | Min / Max |
| Market Cap | Min / Max |
| Socials | Require Twitter / Website / Telegram |

### Supported Launchpads (9+)
1. Pump.fun
2. Bonk
3. MoonIt
4. Believe
5. Jupiter
6. MoonE
7. Boop
8. Launchlabs
9. Meteora (Dynamic)

### Performance Targets
- Page load: < 1 second
- Data refresh: < 100ms
- Up to 30 tokens per section
- Virtualized rendering for scroll performance

---

## 2. Token Trading Page

### Purpose
Full trading interface for individual tokens with real-time data and instant execution.

### Entry Points
- Discovery page token click
- Search bar (token name/ticker/CA)
- Watchlist token click
- Direct link sharing

### Chart Section

**Timeframes (14 total)**:
| Category | Options |
|----------|---------|
| Seconds | 1s, 5s, 15s, 30s |
| Minutes | 1m, 5m, 15m, 30m |
| Hours | 1h, 4h, 6h, 12h |
| Days | 1d, 1w |

**Chart Overlays**:
- Average entry line (your cost basis)
- Active limit order levels
- Take-profit / Stop-loss levels
- Your buy/sell transaction markers
- Tracked wallet activity markers

**Chart Events**:
- Launchpad graduation markers
- Liquidity pool additions

**Performance**:
- Load time: < 0.5 seconds
- No breaks during rapid price movements

### Token Header Info

| Field | Description |
|-------|-------------|
| Token Image | Hover-enabled |
| Bonding Curve % | Progress indicator |
| Token Name | Full name |
| Ticker | Symbol |
| Contract Address | Copyable |
| Social Links | Twitter, Telegram, Discord, Website |
| Age | Time since deployment |
| Current Price | USD value |
| Market Cap | Fully diluted |
| 24h Volume | Trading volume |
| Liquidity | Pool reserves |
| Global Fees Paid | Total swap fees |
| Supply | Total token supply |

### Token Information Panel

| Metric | Description |
|--------|-------------|
| Top 10 Holder % | Concentration in top 10 wallets |
| Token Image | Preview |
| Dev Holdings % | Developer wallet position |
| Bundler % | Coordinated wallet groups |
| Holder Count | Unique holders with balance > 0 |
| Sniper % | Same-slot buyers |
| Insider % | Transfer recipients (no buy tx) |
| Phishing % | Known phishing wallets |
| Dex Paid | DexScreener $299 verification (Y/N) |
| No Mint | Mint authority disabled (Y/N) |
| Burned LP % | LP tokens burned |
| Rug Risk % | Multi-factor risk score |
| Blacklist % | Flagged wallets |

### Developer Information

| Field | Description |
|-------|-------------|
| Dev Wallet Address | Deployer wallet |
| Funding Source | Exchange or wallet (1 hop) |
| Funding Amount | SOL used to fund |
| Funding Age | When funded |

### PnL Module

| Field | Description |
|-------|-------------|
| Bought | Total SOL spent |
| Sold | Total SOL received |
| Holding | Current value |
| Current PnL | SOL + percentage |
| Export Button | Generate PnL card |

### Trading Panel

**Order Types**:
- Market Order: Execute at current price
- Limit Order: Trigger on price/mcap/multiple
- Take-Profit: Auto sell at target
- Stop-Loss: Auto sell at floor

**Market Order**:
- SOL input field
- Preset amount buttons

**Limit Order**:
- Trigger type: Market cap multiple or percentage
- SOL amount specification
- Expiration (optional)

**Transaction Config (per order)**:
- Slippage tolerance
- Transaction fee / tip
- MEV protection toggle

**Presets**:
- 3 savable preset configurations
- Quick switch between presets

**Multi-Wallet Selection** (when enabled):
- Toggle wallets on/off
- Intelligent order distribution
- Example: 5 SOL with 5 wallets = 1 SOL each
- Auto-adjusts for insufficient balances

### Price Change Indicators
- 5-minute change %
- 1-hour change %
- 6-hour change %
- 24-hour change %

### Volume Metrics (adapts to timeframe)
- Buy count
- Sell count
- Buy volume ($)
- Sell volume ($)
- Net buy volume

---

## 3. Tabs

### Trades Tab
Real-time display of transactions on the token.

| Column | Description |
|--------|-------------|
| Age | Time since transaction |
| Type | Buy / Sell |
| Market Cap | At time of trade |
| Amount | Token amount |
| Total | USD or SOL value |
| Trader | Wallet address |

**Features**:
- Filter by specific wallet
- Extend to split-screen view

### Holders Tab
Top 30 holders (preloaded), only wallets with balance > 0.

| Column | Description |
|--------|-------------|
| Rank | Position by holdings |
| Wallet | Address |
| % of Supply | Holdings percentage |
| Current Value | USD value |
| Average Entry | Volume-weighted |
| Realized PnL | From completed sells |
| Total PnL | Realized + unrealized |
| Remaining % | Current vs peak holdings (progress bar) |
| Funding Address | 1 hop trace |
| Funding Amount | SOL |

**Wallet Badges**:
| Badge | Definition |
|-------|------------|
| Sniper | Bought in creation slot |
| Insider | Received via transfer |
| Bundler | Coordinated group |
| Fresh Wallet | <7 days, <10 txs |
| Dev | Deployed or initial supply |
| KOL | Key Opinion Leader |
| Top 10 | In top 10 by balance |

**Filters**:
- You (own holdings)
- Tracking (wallet tracker wallets)
- Token Dev
- Fresh Wallets
- Insiders
- Bundlers
- KOLs

### Positions Tab
User's positions across all tokens.

| Column | Description |
|--------|-------------|
| Token Name | With image |
| Supply % | Your percentage |
| Invested | Total bought |
| Sold | Total sold |
| Market Cap | Current |
| Current Holdings | Value |
| Total PnL | Profit/loss |

### Orders Tab
Active limit orders.

| Column | Description |
|--------|-------------|
| Created At | Timestamp |
| Token | Ticker + image |
| Type | Buy Limit / Sell Limit / TP / SL |
| Status | Placed / Partially Filled / Canceled |
| Trigger Market Cap | Condition |
| Amount | Order size |
| Action | Cancel button |

### History Tab
All user transactions across all tokens.

| Column | Description |
|--------|-------------|
| Date | Transaction date |
| Token | Name |
| Type | Limit Buy / Market Buy / Limit Sell / Market Sell |
| Success | Success / Failed |
| Average Market Cap | At execution |
| Amount | Trade amount |
| Value | USD value |

**Features**:
- Global SOL/USD toggle

### Dev Tokens Tab
Other tokens by current token's developer.

| Column | Description |
|--------|-------------|
| Token | Name |
| Migrated | Y/N (graduated) |
| Current Market Cap | USD |
| Current Liquidity | USD |
| 1h Volume | Recent activity |

---

## 4. Portfolio Page

### Purpose
Unified portfolio management for Spot and Perps.

### Top-Level Controls
- Toggle: Spot / Perps
- Timeframe filter: 1D, 7D, 30D, Max
- SOL/USD toggle

### Spot - Balance Module

| Field | Description |
|-------|-------------|
| Total Value | Portfolio worth (SOL + USD) |
| Available Balance | Liquid SOL |

### Spot - Performance Module

| Field | Description |
|-------|-------------|
| Unrealized PnL | Open positions |
| Realized PnL | Closed positions |

### Performance Categories (Realized PnL Only)

| Category | Definition | Threshold |
|----------|------------|-----------|
| Moonshot | >5x realized gain | >500% |
| Pump | 2-5x realized gain | 100-500% |
| Win | 1-2x realized gain | 0-100% |
| Loss | Realized loss | 0% to -50% |
| Rekt | Major loss | >50% loss |

**Note**: Tokens with no sells don't appear in categories until first sell.

### Spot - Position Tables

**Active Positions**:
| Column | Description |
|--------|-------------|
| Token | Image + name |
| Bought | Total invested |
| Sold | Total received |
| Remaining | Current value |
| PnL | With share button |
| Actions | Hide, Sell |

**Sell Modal**:
- Preset percentages: 25%, 50%, 75%, 100%
- Custom amount input
- Slippage config
- Priority fee config
- Confirm/Cancel

**Closed Positions**:
- Token, Bought, Sold, Time, PnL, Hide action

**Hidden Positions**:
- Still counted in totals
- Can be sold from Hidden tab
- Unhide action available

**Activity Tab**:
- Type (Buy/Sell), Token, Amount, Market Cap, Age
- Filter by token
- SOL/USD toggle

### Perps - Balance Module

| Field | Description |
|-------|-------------|
| All-Time Volume | Total traded |
| All-Time PnL | Cumulative P&L |
| Number of Trades | Trade count |
| Account Value | Current equity |

### Perps - Open Positions

| Column | Description |
|--------|-------------|
| Coin | Asset + leverage (e.g., "BTC 5x") |
| Direction | Long (green) / Short (red) |
| Size | Position size |
| Position Value | Current value |
| Entry Price | Average entry |
| Mark Price | Current market price |
| PnL (ROE%) | Profit/loss + return on equity |
| Liquidation Price | Risk threshold |
| Margin | Allocated collateral |
| Funding | Accumulated payments |
| Actions | Limit Close, Market Close |

### Perps - Trade History

| Column | Description |
|--------|-------------|
| Time | Timestamp |
| Coin | Asset |
| Direction | Long/Short |
| Price | Execution price |
| Size | Trade size |
| Trade Value | USD value |
| Fee | Fee paid |
| Closed PnL | If position closed |

### PnL Share Modal
- Generate shareable image card
- Token name, PnL amount, percentage
- Entry/exit prices (for individual positions)
- Click branding
- Copy to clipboard
- Download as PNG
- Direct share to Twitter

---

## 5. Settings Page

### Trading Settings
- Default slippage
- Buy priority fee
- Sell priority fee
- MEV protection toggle
- Auto-buy on paste (amount)

### UI Presets
- Buy preset amounts (4 configurable)
- Sell preset percentages (4 configurable)

### Notifications
- Price alerts
- Position alerts
- Whale activity

### Multi-Wallet
- View all wallets
- Create / Import
- Archive / Unarchive
- Transfer management

### Account
- Click username
- Referral link
- Connected accounts
- Logout
