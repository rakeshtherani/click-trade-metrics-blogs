# BlockLiquidity - Product Specification

**Hyperliquid Analytics & Whale Intelligence Platform**

---

## Executive Summary

BlockLiquidity is a multi-channel analytics and intelligence platform for Hyperliquid perpetual futures traders. It provides institutional-grade whale tracking, copy trading signals, liquidation monitoring, and risk analytics - all powered by real-time blockchain data.

**Core Philosophy:**
> "See what whales see, trade what whales trade, before everyone else"

BlockLiquidity consolidates the entire trading intelligence workflow (discovery → research → signals → monitoring) into a unified platform across Web, Telegram, and API.

---

## Table of Contents

1. [Target Users](#1-target-users)
2. [Products & Channels](#2-products--channels)
3. [Data Architecture](#3-data-architecture)
4. [Feature Specifications](#4-feature-specifications)
5. [Technical Architecture](#5-technical-architecture)
6. [API Specification](#6-api-specification)
7. [Database Queries](#7-database-queries)
8. [Roadmap](#8-roadmap)
9. [Competitive Position](#9-competitive-position)
10. [Business Model](#10-business-model)

---

## 1. Target Users

| Persona | Description | Primary Channel | Key Needs |
|---------|-------------|-----------------|-----------|
| **Whale Hunter** | Tracks large traders, copies winning strategies | Web Dashboard | Real-time whale alerts, position tracking |
| **Risk Manager** | Monitors portfolio risk, drawdown, VaR | Web Dashboard | Risk metrics, alerts, reporting |
| **Degen Trader** | Fast-moving, needs quick signals | Telegram Bot | Instant alerts, one-tap info |
| **Quant/Algo Trader** | Builds automated strategies | API | Raw data access, webhooks |
| **Fund Manager** | Manages capital, needs analytics | Web Dashboard | Leaderboard, copy signals, risk |
| **Data Analyst** | Research, backtesting | API + Web | Historical data, exports |

---

## 2. Products & Channels

### 2.1 Web Dashboard (Primary Product)

The Web Dashboard is BlockLiquidity's flagship product - a comprehensive analytics terminal for Hyperliquid traders.

#### 2.1.1 Home / Market Overview

**Purpose:** Bird's eye view of the entire Hyperliquid market

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  BLOCKLIQUIDITY                          [Search] [Alerts 🔔] [Settings ⚙️] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌────────────┐│
│  │ TOTAL OI        │ │ 24H VOLUME      │ │ 24H LIQUIDATIONS│ │ NET FLOW   ││
│  │ $4.2B          │ │ $12.8B          │ │ $156M           │ │ +$23M      ││
│  │ ↑ 2.3%         │ │ ↑ 15.2%         │ │ 65% LONGS       │ │ INFLOW     ││
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ MARKET TABLE                                              [Filter ▼]   ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Token │ Price    │ 24h %  │ Funding │ OI       │ L/S Ratio │ Liq 24h  ││
│  │ ──────┼──────────┼────────┼─────────┼──────────┼───────────┼──────────││
│  │ BTC   │ $104,521 │ +1.2%  │ +0.010% │ $1.2B    │ 1.45 LONG │ $12.3M   ││
│  │ ETH   │ $2,972   │ -0.5%  │ +0.021% │ $890M    │ 1.51 LONG │ $8.7M    ││
│  │ HYPE  │ $21.38   │ -8.2%  │ -0.052% │ $156M    │ 1.36 LONG │ $45.2M   ││
│  │ SOL   │ $169.42  │ +2.1%  │ +0.015% │ $234M    │ 1.92 LONG │ $5.1M    ││
│  │ ...   │          │        │         │          │           │          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌──────────────────────────────┐ ┌────────────────────────────────────────┐│
│  │ 🐋 TOP WHALE MOVES (Live)    │ │ 💥 RECENT LIQUIDATIONS                 ││
│  │ ────────────────────────────│ │ ──────────────────────────────────────││
│  │ • Wallet 6071 SHORT HYPE    │ │ • LONG BTC $2.3M liquidated @ $103.2K ││
│  │   +$26.7M PnL               │ │ • SHORT ETH $890K liquidated @ $2,985 ││
│  │ • Wallet 15177 SHORT HYPE   │ │ • LONG HYPE $1.2M liquidated @ $20.50 ││
│  │   +$15M PnL                 │ │ • ...                                 ││
│  │ • Wallet 1914 LONG ETH      │ │                                       ││
│  │   +$2M PnL                  │ │ [View All Liquidations →]             ││
│  │ [View All Whales →]         │ │                                       ││
│  └──────────────────────────────┘ └────────────────────────────────────────┘│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Market Table Columns:**

| Column | Source | Description |
|--------|--------|-------------|
| Token | `perp_tokens.token` | Trading pair symbol |
| Price | `perp_tokens.markPx` | Current mark price |
| 24h % | Derived | Price change percentage |
| Funding | `perp_tokens.funding` | 8-hour funding rate |
| OI | `perp_tokens.openInterest` | Total open interest |
| L/S Ratio | `perp_asset_positions_v2` | Long/Short ratio |
| Liq 24h | `hl_liquidations` | Liquidation volume |
| Volume | `perp_tokens.dayNtlVlm` | 24h trading volume |

**Sorting Options:**
- By Open Interest (default)
- By Volume
- By Funding Rate
- By Liquidations
- By Price Change

**Filtering:**
- Min OI threshold
- Funding rate range
- Only showing tokens with liquidations

---

#### 2.1.2 Whale Tracker Page

**Purpose:** Track and analyze large trader positions

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🐋 WHALE TRACKER                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FILTER WHALES                                                          ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Min Position: [$100K ▼]  Strategy: [All ▼]  Token: [All ▼]            ││
│  │ PnL Status: [All ▼]  Sort By: [Total PnL ▼]                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TOP WHALES BY PNL                                                      ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Rank│ Wallet  │ Exposure │ Unrealized │ Tokens│ Bias    │ Win Rate    ││
│  │ ────┼─────────┼──────────┼────────────┼───────┼─────────┼─────────────││
│  │ 1   │ 6071    │ $92M     │ +$32M      │ 7     │ 100% 📉 │ 85%         ││
│  │ 2   │ 15177   │ $45M     │ +$18M      │ 3     │ 90% 📉  │ 78%         ││
│  │ 3   │ 4259    │ $38M     │ +$12M      │ 4     │ 95% 📉  │ 82%         ││
│  │ 4   │ 1914    │ $25M     │ +$5M       │ 2     │ 80% 📈  │ 71%         ││
│  │ ... │         │          │            │       │         │             ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ WHALE SPOTLIGHT: Wallet 6071                           [Follow] [Alert]││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  Strategy: PERMA-BEAR (100% Short)     Total PnL: +$32,145,890        ││
│  │  Active Since: 2024-01-15              Win Rate: 85%                  ││
│  │  Avg Position Size: $13.1M             Sharpe: 2.4                    ││
│  │                                                                        ││
│  │  CURRENT POSITIONS:                                                    ││
│  │  ┌────────┬───────────┬────────────┬─────────────┬──────────────────┐ ││
│  │  │ Token  │ Direction │ Size       │ Entry       │ Unrealized PnL   │ ││
│  │  ├────────┼───────────┼────────────┼─────────────┼──────────────────┤ ││
│  │  │ HYPE   │ SHORT 📉  │ $27.8M     │ $42.00      │ +$26.7M (+96%)   │ ││
│  │  │ ETH    │ SHORT 📉  │ $39.4M     │ $3,240      │ +$3.7M (+9.4%)   │ ││
│  │  │ BTC    │ SHORT 📉  │ $11.7M     │ $108,500    │ +$300K (+2.6%)   │ ││
│  │  │ FART   │ SHORT 📉  │ $1.1M      │ $1.85       │ +$1.1M (+100%)   │ ││
│  │  │ INJ    │ SHORT 📉  │ $170K      │ $28.50      │ +$312K (+183%)   │ ││
│  │  │ SOL    │ SHORT 📉  │ $660K      │ $172.30     │ +$28K (+4.2%)    │ ││
│  │  │ XRP    │ SHORT 📉  │ $11.1M     │ $3.12       │ -$55K (-0.5%)    │ ││
│  │  └────────┴───────────┴────────────┴─────────────┴──────────────────┘ ││
│  │                                                                        ││
│  │  RECENT ACTIVITY:                                                      ││
│  │  • 2h ago: Increased HYPE short by $5M                                ││
│  │  • 6h ago: Opened new XRP short $11.1M                                ││
│  │  • 1d ago: Closed SOL short +$450K profit                             ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

**Whale Classification System:**

| Classification | Criteria | Badge |
|----------------|----------|-------|
| Mega Whale | Position > $10M | 🐋🐋 |
| Whale | Position $1M - $10M | 🐋 |
| Dolphin | Position $100K - $1M | 🐬 |
| Shark | High win rate (>70%) + profitable | 🦈 |
| Perma-Bull | >80% long positions | 📈 |
| Perma-Bear | >80% short positions | 📉 |
| Degen | High leverage (>20x avg) | 🎰 |
| Sniper | Quick trades (<1h avg hold) | 🎯 |

**Whale Metrics:**

| Metric | Calculation | Description |
|--------|-------------|-------------|
| Total Exposure | SUM(ABS(position_value)) | Total position size |
| Net Exposure | SUM(position_value) | Long minus short |
| Unrealized PnL | SUM(unrealized_pnl) | Paper profit/loss |
| Win Rate | Winning trades / Total trades | Historical accuracy |
| Avg Hold Time | AVG(close_time - open_time) | Position duration |
| Sharpe Ratio | From wallet_metrics_v2 | Risk-adjusted return |
| Bias | Long% vs Short% | Directional tendency |

---

#### 2.1.3 Token Deep Dive Page

**Purpose:** Comprehensive analysis for individual tokens

**Entry Points:**
- Click token from Market Overview
- Search bar
- Direct URL: `/token/HYPE`

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  HYPE-PERP                                           [Add to Watchlist ⭐]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────┐ ┌──────────────────────────────┐│
│  │         PRICE CHART                    │ │ TOKEN METRICS                ││
│  │                                        │ │ ────────────────────────────││
│  │    [1m][5m][15m][1h][4h][1d][1w]      │ │ Price: $21.38                ││
│  │                                        │ │ 24h Change: -8.2% 📉         ││
│  │         📈 TradingView Chart           │ │ Mark/Oracle: -0.12%         ││
│  │                                        │ │                              ││
│  │                                        │ │ Open Interest: $156M         ││
│  │                                        │ │ OI Change 24h: -5.4%         ││
│  │                                        │ │                              ││
│  │                                        │ │ Funding (8h): -0.052%        ││
│  │                                        │ │ Funding Ann: -57% 📉         ││
│  │                                        │ │                              ││
│  │                                        │ │ 24h Volume: $892M            ││
│  │                                        │ │ 24h Liquidations: $45.2M    ││
│  └────────────────────────────────────────┘ └──────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ POSITIONING ANALYSIS                                                    ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  Long/Short Ratio: 1.36          Crowding Score: +0.15 (Slightly Long)││
│  │                                                                        ││
│  │  ┌─────────────────────────────────────────────────────────────────┐  ││
│  │  │ LONGS ████████████████████░░░░░░░░░░░░ SHORTS                   │  ││
│  │  │       58% ($90.5M)              42% ($65.5M)                     │  ││
│  │  └─────────────────────────────────────────────────────────────────┘  ││
│  │                                                                        ││
│  │  Whale Sentiment: BEARISH (Top 10 wallets: 72% short)                 ││
│  │  Retail Sentiment: BULLISH (Small wallets: 64% long)                  ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TABS: [Whales] [Liquidations] [Orderbook] [Trades] [Funding History]   ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  TOP WHALE POSITIONS (HYPE):                                          ││
│  │  ┌────────┬───────────┬────────────┬─────────────┬──────────────────┐ ││
│  │  │ Wallet │ Direction │ Size       │ Entry       │ Unrealized PnL   │ ││
│  │  ├────────┼───────────┼────────────┼─────────────┼──────────────────┤ ││
│  │  │ 6071   │ SHORT 📉  │ $27.8M     │ $42.00      │ +$26.7M (+96%)   │ ││
│  │  │ 323970 │ LONG 📈   │ $29.5M     │ $38.67      │ -$23.8M (-81%)   │ ││
│  │  │ 15177  │ SHORT 📉  │ $15.0M     │ $42.87      │ +$15.0M (+100%)  │ ││
│  │  │ 4259   │ SHORT 📉  │ $30.0M     │ $28.67      │ +$10.2M (+34%)   │ ││
│  │  │ 2325   │ LONG 📈   │ $4.7M      │ $37.74      │ -$3.6M (-77%)    │ ││
│  │  └────────┴───────────┴────────────┴─────────────┴──────────────────┘ ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

**Token Metrics Panel:**

| Metric | Source | Update Frequency |
|--------|--------|------------------|
| Price | `perp_tokens.markPx` | Real-time |
| 24h Change | Derived | 1 minute |
| Mark/Oracle Spread | `perp_tokens` | Real-time |
| Open Interest | `perp_tokens.openInterest` | Real-time |
| Funding Rate | `perp_tokens.funding` | 8 hours |
| Volume | `perp_tokens.dayNtlVlm` | Real-time |
| Liquidations | `hl_liquidations` | Real-time |
| L/S Ratio | `perp_asset_positions_v2` | 5 minutes |
| Crowding Score | Derived | 5 minutes |

**Tabs Content:**

| Tab | Content | Source |
|-----|---------|--------|
| Whales | Top positions by size | `perp_asset_positions_v2` |
| Liquidations | Recent liquidations | `hl_liquidations` |
| Orderbook | Depth & imbalance | `hl_orderbook` |
| Trades | Recent large trades | `hl_trades` |
| Funding | Historical funding | `perp_tokens` |

---

#### 2.1.4 Liquidation Monitor Page

**Purpose:** Real-time liquidation tracking and cascade detection

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  💥 LIQUIDATION MONITOR                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌────────────┐│
│  │ 24H LIQUIDATIONS│ │ LONGS REKT      │ │ SHORTS REKT     │ │ CASCADE    ││
│  │ $156.2M        │ │ $101.4M (65%)   │ │ $54.8M (35%)    │ │ ⚠️ MEDIUM   ││
│  │ ↑ 234%         │ │ 1,245 positions │ │ 678 positions   │ │            ││
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LIQUIDATION FEED (Live)                              [Pause] [Filter]  ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Time     │ Token │ Side  │ Size      │ Price      │ Wallet            ││
│  │ ─────────┼───────┼───────┼───────────┼────────────┼───────────────────││
│  │ 2s ago   │ HYPE  │ LONG  │ $1.23M    │ $20.45     │ 0x8f3...2a1      ││
│  │ 15s ago  │ BTC   │ LONG  │ $892K     │ $103,250   │ 0x2d1...8c4      ││
│  │ 23s ago  │ HYPE  │ LONG  │ $567K     │ $20.52     │ 0x9a2...3b5      ││
│  │ 45s ago  │ ETH   │ SHORT │ $234K     │ $2,985     │ 0x1c3...7d2      ││
│  │ 1m ago   │ HYPE  │ LONG  │ $2.1M     │ $20.78     │ 0x4e5...9f1      ││
│  │ ...      │       │       │           │            │                   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌──────────────────────────────┐ ┌────────────────────────────────────────┐│
│  │ LIQUIDATION HEATMAP         │ │ AT-RISK POSITIONS                      ││
│  │ ────────────────────────────│ │ ──────────────────────────────────────││
│  │                              │ │ Positions within 5% of liquidation:   ││
│  │  Token  │ Longs  │ Shorts   │ │                                        ││
│  │  ───────┼────────┼──────────│ │ Token │ Side │ Value    │ Distance    ││
│  │  HYPE   │ $45.2M │ $8.3M    │ │ ──────┼──────┼──────────┼─────────────││
│  │  BTC    │ $12.3M │ $5.1M    │ │ HYPE  │ LONG │ $2.3M    │ 2.1%        ││
│  │  ETH    │ $8.7M  │ $3.2M    │ │ BTC   │ LONG │ $1.8M    │ 3.4%        ││
│  │  SOL    │ $5.1M  │ $2.4M    │ │ ETH   │ SHORT│ $950K    │ 4.2%        ││
│  │  ...    │        │          │ │ ...   │      │          │             ││
│  └──────────────────────────────┘ └────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ CASCADE DETECTION                                                       ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │ ⚠️ ACTIVE CASCADE DETECTED: HYPE                                       ││
│  │                                                                        ││
│  │ • 15 liquidations in last 5 minutes                                   ││
│  │ • Total value: $8.7M                                                  ││
│  │ • Price impact: -4.2%                                                 ││
│  │ • Risk: More liquidations likely at $19.80 level                      ││
│  │                                                                        ││
│  │ Liquidation Clusters:                                                  ││
│  │ • $19.50 - $20.00: $12.3M in long positions at risk                  ││
│  │ • $18.00 - $18.50: $8.9M in long positions at risk                   ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

**Cascade Alert Levels:**

| Level | Criteria | UI Indicator |
|-------|----------|--------------|
| LOW | <5 liquidations in 5 min | 🟢 Green |
| MEDIUM | 5-10 liquidations in 5 min | 🟡 Yellow |
| HIGH | 10-20 liquidations in 5 min | 🟠 Orange |
| EXTREME | >20 liquidations in 5 min | 🔴 Red + Alert |

---

#### 2.1.5 Leaderboard Page

**Purpose:** Discover and track top performing traders

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🏆 LEADERBOARD                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TIME PERIOD: [24H] [7D] [30D] [All Time]    SORT BY: [PnL ▼]          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TOP TRADERS BY PNL (30D)                                               ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Rank│ Wallet     │ PnL        │ Volume    │ Trades│ Win%  │ Sharpe   ││
│  │ ────┼────────────┼────────────┼───────────┼───────┼───────┼──────────││
│  │ 🥇 1│ 0x8f3...2a │ +$32.1M    │ $456M     │ 234   │ 85%   │ 2.4      ││
│  │ 🥈 2│ 0x2d1...8c │ +$18.5M    │ $312M     │ 189   │ 78%   │ 1.9      ││
│  │ 🥉 3│ 0x9a2...3b │ +$12.3M    │ $234M     │ 156   │ 82%   │ 2.1      ││
│  │   4 │ 0x1c3...7d │ +$8.9M     │ $178M     │ 123   │ 71%   │ 1.6      ││
│  │   5 │ 0x4e5...9f │ +$6.2M     │ $145M     │ 98    │ 75%   │ 1.8      ││
│  │ ... │            │            │           │       │       │          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TRADER PROFILE: 0x8f3...2a1                      [Follow] [Copy Alert]││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  STATS (30D):                          RISK METRICS:                   ││
│  │  • Total PnL: +$32.1M                  • Sharpe Ratio: 2.4            ││
│  │  • Win Rate: 85%                       • Sortino Ratio: 3.1           ││
│  │  • Avg Win: +$892K                     • Max Drawdown: -8.2%          ││
│  │  • Avg Loss: -$234K                    • VaR (95%): -$1.2M            ││
│  │  • Profit Factor: 4.2                  • Avg Leverage: 5.2x           ││
│  │                                                                        ││
│  │  CURRENT POSITIONS:                                                    ││
│  │  [Same as Whale Tracker detail view]                                  ││
│  │                                                                        ││
│  │  TRADE HISTORY:                                                        ││
│  │  [Table of recent trades with PnL]                                    ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

**Leaderboard Metrics:**

| Metric | Source | Description |
|--------|--------|-------------|
| Realized PnL | `hl_user_fills.closed_pnl` | Actual profits |
| Volume | `hl_user_fills` | Total traded |
| Trade Count | `hl_user_fills` | Number of trades |
| Win Rate | Derived | Winning trades % |
| Sharpe Ratio | `wallet_metrics_v2` | Risk-adjusted return |
| Sortino Ratio | `wallet_metrics_v2` | Downside-adjusted return |
| Max Drawdown | `maximum_drawdown_daily` | Worst decline |

---

#### 2.1.6 Capital Flows Page

**Purpose:** Track money flowing in/out of Hyperliquid

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  💰 CAPITAL FLOWS                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌────────────┐│
│  │ 24H DEPOSITS    │ │ 24H WITHDRAWALS │ │ NET FLOW        │ │ 7D TREND   ││
│  │ $45.2M         │ │ $22.1M          │ │ +$23.1M         │ │ ↑ INFLOW   ││
│  │ 1,234 txns     │ │ 567 txns        │ │ BULLISH         │ │ +$89M      ││
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ NET FLOW CHART (30 Days)                                               ││
│  │                                                                        ││
│  │     📈 [Chart showing daily net flow bars - green/red]                 ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌──────────────────────────────┐ ┌────────────────────────────────────────┐│
│  │ WHALE DEPOSITS (>$100K)     │ │ WHALE WITHDRAWALS (>$100K)             ││
│  │ ────────────────────────────│ │ ──────────────────────────────────────││
│  │ Time   │ Wallet  │ Amount   │ │ Time   │ Wallet  │ Amount             ││
│  │ ───────┼─────────┼──────────│ │ ───────┼─────────┼────────────────────││
│  │ 2h ago │ 0x8f3.. │ $2.3M    │ │ 4h ago │ 0x2d1.. │ $1.8M              ││
│  │ 5h ago │ 0x9a2.. │ $890K    │ │ 8h ago │ 0x1c3.. │ $567K              ││
│  │ 8h ago │ 0x4e5.. │ $456K    │ │ 1d ago │ 0x7b2.. │ $234K              ││
│  │ ...    │         │          │ │ ...    │         │                    ││
│  └──────────────────────────────┘ └────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FLOW INTERPRETATION                                                     ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │ 📊 Current Status: ACCUMULATION PHASE                                  ││
│  │                                                                        ││
│  │ • Net inflows for 5 consecutive days                                  ││
│  │ • Whale deposits increasing (+34% vs last week)                       ││
│  │ • Withdrawal rate decreasing (-12% vs last week)                      ││
│  │                                                                        ││
│  │ Historical Pattern Match: Similar to Oct 2024 (preceded 40% rally)    ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### 2.1.7 Risk Analytics Page

**Purpose:** Portfolio risk monitoring and analysis

**Layout:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  📊 RISK ANALYTICS                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ MARKET-WIDE RISK INDICATORS                                            ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      ││
│  │  │ FEAR/GREED  │ │ FUNDING     │ │ LEVERAGE    │ │ LIQ RISK    │      ││
│  │  │ 72 GREED    │ │ +0.015%     │ │ 8.2x AVG    │ │ MEDIUM      │      ││
│  │  │ 🟡          │ │ NEUTRAL     │ │ HIGH        │ │ 🟡          │      ││
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘      ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TOP TRADERS RISK METRICS                                               ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │ Wallet  │ Sharpe │ Sortino│ VaR 95%  │ CVaR    │ Max DD  │ Grade     ││
│  │ ────────┼────────┼────────┼──────────┼─────────┼─────────┼───────────││
│  │ 0x8f3.. │ 2.4    │ 3.1    │ -$1.2M   │ -$1.8M  │ -8.2%   │ A+        ││
│  │ 0x2d1.. │ 1.9    │ 2.4    │ -$890K   │ -$1.3M  │ -12.1%  │ A         ││
│  │ 0x9a2.. │ 1.6    │ 2.1    │ -$567K   │ -$890K  │ -15.4%  │ B+        ││
│  │ ...     │        │        │          │         │         │           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ UNREALIZED PNL BY TIER                                                 ││
│  │ ───────────────────────────────────────────────────────────────────────││
│  │                                                                        ││
│  │  Tier           │ Traders │ Long PnL   │ Short PnL │ Net        │ Status││
│  │  ────────────────┼─────────┼────────────┼───────────┼────────────┼──────││
│  │  Blue Whale II  │ 333     │ -$4.3M     │ +$44.6M   │ +$40.3M    │ 📈   ││
│  │  Blue Whale I   │ 532     │ -$3.3M     │ +$30.9M   │ +$27.6M    │ 📈   ││
│  │  Large Whale I  │ 468     │ -$11.3M    │ +$21.5M   │ +$10.3M    │ 📈   ││
│  │  Minnow I       │ 6,506   │ -$56K      │ +$17K     │ -$39K      │ 📉   ││
│  │  ...            │         │            │           │            │      ││
│  │                                                                        ││
│  │  INSIGHT: Whales are SHORT and profitable, Retail is LONG and losing  ││
│  │                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### 2.1.8 Analytics & Insights Page

**Purpose:** Advanced analytics and market insights

**Sections:**

1. **Time-of-Day Analysis**
   - Hourly volume profile
   - Best trading hours
   - Session analysis (Asia/Europe/US)

2. **Correlation Matrix**
   - Token price correlations
   - Funding rate correlations

3. **Technical Indicators**
   - RSI by token
   - Bollinger Bands
   - Moving averages

4. **Copy Trading Detection**
   - Correlated wallets
   - Leader-follower relationships

5. **Cross-Exchange Comparison**
   - HL vs Binance spreads
   - HL vs Coinbase spreads
   - Arbitrage opportunities

---

### 2.2 Telegram Bot

**Purpose:** Instant alerts and quick lookups via Telegram

**Bot Name:** `@BlockLiquidityBot`

#### Commands:

| Command | Function | Example |
|---------|----------|---------|
| `/start` | Welcome + dashboard | - |
| `/whale` | Top whale positions | `/whale` |
| `/whale [wallet_id]` | Specific whale detail | `/whale 6071` |
| `/token [symbol]` | Token overview | `/token HYPE` |
| `/liq` | Recent liquidations | `/liq` |
| `/liq [symbol]` | Token liquidations | `/liq HYPE` |
| `/funding` | Funding rates | `/funding` |
| `/flows` | Capital flows | `/flows` |
| `/alert` | Manage alerts | `/alert` |
| `/leaderboard` | Top traders | `/leaderboard` |
| `/help` | Command list | `/help` |

#### Alert Types:

| Alert | Trigger | Example Message |
|-------|---------|-----------------|
| Whale Move | Whale opens/closes position | "🐋 Wallet 6071 opened SHORT HYPE $5M @ $21.50" |
| Liquidation Cascade | >5 liquidations in 5 min | "💥 CASCADE: 12 HYPE longs liquidated ($8.7M)" |
| Funding Extreme | Funding > 0.1% or < -0.1% | "⚠️ HYPE funding at -0.15% (extremely negative)" |
| Price Alert | User-defined price levels | "🔔 HYPE hit your target: $20.00" |
| Whale Follow | Followed whale trades | "🐋 Your followed whale 6071 sold ETH" |

#### Sample Bot Interactions:

```
User: /whale 6071

Bot:
┌─────────────────────────────────────┐
│ 🐋 WHALE PROFILE: 6071              │
├─────────────────────────────────────┤
│ Strategy: PERMA-BEAR (100% Short)   │
│ Total Exposure: $92M                │
│ Unrealized PnL: +$32.1M             │
│ Win Rate: 85%                       │
│                                     │
│ Top Positions:                      │
│ • HYPE SHORT $27.8M (+$26.7M) ✅    │
│ • ETH SHORT $39.4M (+$3.7M) ✅      │
│ • BTC SHORT $11.7M (+$300K) ✅      │
│                                     │
│ [Follow Alerts] [Full Profile]      │
└─────────────────────────────────────┘
```

```
User: /token HYPE

Bot:
┌─────────────────────────────────────┐
│ HYPE-PERP                           │
├─────────────────────────────────────┤
│ Price: $21.38 (-8.2%)               │
│ Funding (8h): -0.052%               │
│ OI: $156M (-5.4%)                   │
│ L/S Ratio: 1.36 (LONG bias)         │
│ 24h Liq: $45.2M (65% longs)         │
│                                     │
│ Whale Sentiment: BEARISH 📉         │
│ Top 3 whales: 100% SHORT            │
│                                     │
│ [View Whales] [Liquidations] [Chart]│
└─────────────────────────────────────┘
```

---

### 2.3 REST API

**Purpose:** Programmatic access for quants and developers

**Base URL:** `https://api.blockliquidity.io/v1`

#### Endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/markets` | GET | All token metrics |
| `/markets/{symbol}` | GET | Single token detail |
| `/whales` | GET | Top whale positions |
| `/whales/{wallet_id}` | GET | Whale profile |
| `/liquidations` | GET | Recent liquidations |
| `/liquidations/stream` | WS | Real-time liquidations |
| `/leaderboard` | GET | Top traders |
| `/flows` | GET | Capital flows |
| `/risk/{wallet_id}` | GET | Wallet risk metrics |
| `/analytics/time-of-day` | GET | Time analysis |
| `/analytics/correlations` | GET | Token correlations |

#### Authentication:
- API Key in header: `X-API-Key: your_key`
- Rate limits: 100 req/min (free), 1000 req/min (pro)

#### Sample Response:

```json
GET /v1/whales/6071

{
  "wallet_id": 6071,
  "address": "0x8f3...2a1",
  "classification": "MEGA_WHALE",
  "strategy": "PERMA_BEAR",
  "metrics": {
    "total_exposure": 92000000,
    "net_exposure": -92000000,
    "unrealized_pnl": 32145890,
    "win_rate": 0.85,
    "sharpe_ratio": 2.4,
    "avg_leverage": 3.2
  },
  "positions": [
    {
      "token": "HYPE",
      "direction": "SHORT",
      "size_usd": 27800000,
      "entry_price": 42.00,
      "current_price": 21.38,
      "unrealized_pnl": 26700000,
      "pnl_pct": 96.0
    },
    ...
  ],
  "updated_at": "2026-01-21T10:30:00Z"
}
```

---

### 2.4 Webhook Alerts

**Purpose:** Push notifications to user systems

**Supported Destinations:**
- Discord
- Slack
- Telegram
- Custom HTTP endpoint

**Event Types:**
- `whale.position.opened`
- `whale.position.closed`
- `liquidation.cascade`
- `funding.extreme`
- `price.alert`

---

## 3. Data Architecture

### 3.1 Available Data Sources

| Table | Records | Update Frequency | Key Metrics |
|-------|---------|------------------|-------------|
| `perp_tokens` | 4.4B | Real-time | Price, funding, OI, volume |
| `perp_asset_positions_v2` | 2.7B | 5 min | Wallet positions, PnL |
| `spot_tokens` | 4.5B | Real-time | Spot prices, volume |
| `hl_trades` | 390M | Real-time | Trade history |
| `hl_user_fills` | 1.8B | Real-time | User trade fills |
| `hl_liquidations` | 15.9M | Real-time | Liquidation events |
| `hl_orderbook` | - | Real-time | Order depth |
| `wallet_metrics_v2` | - | Daily | Risk metrics |
| `hl_unrealized_pnl_per_tier` | 19.1M | Hourly | PnL by tier |
| `hl_unrealized_pnl_per_coin` | 2.7M | Hourly | PnL by token |
| `hl_deposits` | 829K | Real-time | Deposits |
| `hl_withdraws` | 660K | Real-time | Withdrawals |
| `leaderboard` | 367M | Daily | Top traders |
| `hft_count_daily` | - | Daily | HFT activity |
| `bn_orderbook` | 14M | Real-time | Binance depth |
| `cb_orderbook` | 13.6M | Real-time | Coinbase depth |

### 3.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BLOCKLIQUIDITY DATA FLOW                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │ Hyperliquid  │    │   Binance    │    │  Coinbase    │                  │
│  │   Indexer    │    │   Indexer    │    │   Indexer    │                  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                  │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                      TIMESCALEDB                                │       │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐│       │
│  │  │ perp_tokens │ │ positions   │ │ liquidations│ │ orderbooks ││       │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └────────────┘│       │
│  └─────────────────────────────────────────────────────────────────┘       │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                      BACKEND API                                │       │
│  │  • Query Engine     • WebSocket Server    • Alert Engine        │       │
│  │  • Cache Layer      • Aggregation Jobs    • Webhook Dispatcher  │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │ Web Dashboard│    │ Telegram Bot │    │   REST API   │                  │
│  └─────────────┘    └──────────────┘    └──────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Feature Specifications

### 4.1 Whale Classification Algorithm

```python
def classify_whale(wallet_data):
    exposure = sum(abs(p.value) for p in wallet_data.positions)
    long_pct = sum(p.value for p in wallet_data.positions if p.value > 0) / exposure
    short_pct = 1 - long_pct
    avg_hold_time = calculate_avg_hold_time(wallet_data.trades)
    win_rate = wallet_data.metrics.win_rate

    classifications = []

    # Size classification
    if exposure > 10_000_000:
        classifications.append("MEGA_WHALE")
    elif exposure > 1_000_000:
        classifications.append("WHALE")
    elif exposure > 100_000:
        classifications.append("DOLPHIN")

    # Strategy classification
    if long_pct > 0.8:
        classifications.append("PERMA_BULL")
    elif short_pct > 0.8:
        classifications.append("PERMA_BEAR")

    # Skill classification
    if win_rate > 0.7 and wallet_data.metrics.pnl > 0:
        classifications.append("SHARK")

    # Style classification
    if avg_hold_time < timedelta(hours=1):
        classifications.append("SNIPER")
    if wallet_data.metrics.avg_leverage > 20:
        classifications.append("DEGEN")

    return classifications
```

### 4.2 Cascade Detection Algorithm

```python
def detect_cascade(liquidations, window_minutes=5):
    recent = liquidations.filter(
        timestamp >= now() - timedelta(minutes=window_minutes)
    )

    by_token = recent.group_by('token')

    cascades = []
    for token, liqs in by_token:
        count = len(liqs)
        volume = sum(l.value for l in liqs)

        if count >= 20:
            severity = "EXTREME"
        elif count >= 10:
            severity = "MAJOR"
        elif count >= 5:
            severity = "CASCADE"
        else:
            continue

        cascades.append({
            "token": token,
            "count": count,
            "volume": volume,
            "severity": severity,
            "direction": "LONGS" if sum(1 for l in liqs if l.side == "LONG") > count/2 else "SHORTS"
        })

    return cascades
```

### 4.3 Copy Trading Signal Detection

```python
def detect_copy_trading(trades, time_window=60):
    """Detect wallets that consistently trade within 60 seconds of each other"""

    pairs = []
    for t1 in trades:
        for t2 in trades:
            if t1.wallet != t2.wallet and t1.token == t2.token:
                time_diff = abs(t1.timestamp - t2.timestamp).seconds
                if time_diff <= time_window and t1.side == t2.side:
                    pairs.append((t1.wallet, t2.wallet, t1.token, time_diff))

    # Group by wallet pairs
    pair_counts = Counter((p[0], p[1]) for p in pairs)

    # Return pairs with >5 correlated trades
    return [(w1, w2, count) for (w1, w2), count in pair_counts.items() if count > 5]
```

---

## 5. Technical Architecture

### 5.1 Infrastructure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      BLOCKLIQUIDITY INFRASTRUCTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │ DATABASE LAYER                                                  │       │
│  │ ─────────────────────────────────────────────────────────────── │       │
│  │ • TimescaleDB Primary (existing)                                │       │
│  │ • TimescaleDB Standby (existing)                                │       │
│  │ • Redis (cache + sessions)                                      │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │ BACKEND LAYER                                                   │       │
│  │ ─────────────────────────────────────────────────────────────── │       │
│  │ • API Server (Node.js/FastAPI)                                  │       │
│  │ • WebSocket Server (real-time updates)                          │       │
│  │ • Alert Engine (event processing)                               │       │
│  │ • Telegram Bot Server                                           │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │ FRONTEND LAYER                                                  │       │
│  │ ─────────────────────────────────────────────────────────────── │       │
│  │ • Next.js Web App                                               │       │
│  │ • Hosted on Vercel/CloudFlare                                   │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Tech Stack

| Layer | Technology | Reason |
|-------|------------|--------|
| **Database** | TimescaleDB | Already have it with all data |
| **Cache** | Redis | Fast lookups, rate limiting |
| **Backend API** | FastAPI (Python) | Great for data processing |
| **WebSocket** | Socket.io / FastAPI WS | Real-time updates |
| **Frontend** | Next.js 14 + React | SSR, fast, great DX |
| **Charts** | TradingView Lightweight | Industry standard |
| **Styling** | Tailwind CSS | Dark mode, rapid development |
| **Telegram** | python-telegram-bot | Mature, well-documented |
| **Hosting** | Vercel (FE) + AWS (BE) | Scalable, reliable |

### 5.3 Performance Targets

| Metric | Target |
|--------|--------|
| API Response Time | <200ms |
| WebSocket Latency | <100ms |
| Page Load Time | <2 seconds |
| Data Freshness | <5 seconds |
| Uptime | 99.9% |

---

## 6. API Specification

### 6.1 REST Endpoints

#### Markets

```
GET /v1/markets
Response: List of all tokens with current metrics

GET /v1/markets/{symbol}
Response: Detailed token data including positions breakdown

GET /v1/markets/{symbol}/positions
Response: All positions for a token (paginated)

GET /v1/markets/{symbol}/liquidations
Response: Recent liquidations for token
```

#### Whales

```
GET /v1/whales
Query params: min_exposure, strategy, sort_by, limit
Response: List of whale wallets

GET /v1/whales/{wallet_id}
Response: Full whale profile with positions

GET /v1/whales/{wallet_id}/history
Response: Trade history for wallet

GET /v1/whales/{wallet_id}/pnl
Response: PnL breakdown over time
```

#### Liquidations

```
GET /v1/liquidations
Query params: token, side, min_value, limit
Response: Recent liquidations

GET /v1/liquidations/cascades
Response: Active and recent cascades

WS /v1/liquidations/stream
Real-time liquidation feed
```

#### Leaderboard

```
GET /v1/leaderboard
Query params: period (24h/7d/30d/all), sort_by, limit
Response: Top traders with metrics
```

#### Flows

```
GET /v1/flows
Query params: period, wallet_size
Response: Deposit/withdrawal aggregates

GET /v1/flows/whales
Response: Large wallet movements
```

#### Analytics

```
GET /v1/analytics/time-of-day
Response: Volume and volatility by hour

GET /v1/analytics/correlations
Response: Token correlation matrix

GET /v1/analytics/funding-history/{symbol}
Response: Historical funding rates
```

### 6.2 WebSocket Events

```javascript
// Connect
ws = new WebSocket('wss://api.blockliquidity.io/v1/stream')

// Subscribe to events
ws.send({
  "action": "subscribe",
  "channels": ["liquidations", "whale_moves", "cascades"]
})

// Incoming events
{
  "event": "liquidation",
  "data": {
    "token": "HYPE",
    "side": "LONG",
    "size": 1230000,
    "price": 20.45,
    "wallet": "0x8f3...2a1",
    "timestamp": "2026-01-21T10:30:00Z"
  }
}

{
  "event": "whale_move",
  "data": {
    "wallet_id": 6071,
    "action": "OPEN",
    "token": "ETH",
    "direction": "SHORT",
    "size": 5000000,
    "price": 2950.00,
    "timestamp": "2026-01-21T10:31:00Z"
  }
}

{
  "event": "cascade",
  "data": {
    "token": "HYPE",
    "severity": "MAJOR",
    "count": 15,
    "volume": 8700000,
    "direction": "LONGS",
    "timestamp": "2026-01-21T10:32:00Z"
  }
}
```

---

## 7. Database Queries

### 7.1 Core Queries (Ready to Use)

All queries from **BLOCKLIQUIDITY_METRICS_CATALOG.md** are available, including:

- Market Overview queries
- Whale tracking queries
- Liquidation analysis
- Capital flow queries
- Risk metrics queries
- Time-of-day analysis
- Copy trading detection

### 7.2 Key Materialized Views (Recommended)

```sql
-- Top Whales (refresh every 5 minutes)
CREATE MATERIALIZED VIEW mv_top_whales AS
SELECT
    wallet_id,
    COUNT(DISTINCT token) AS tokens_traded,
    SUM(ABS(position_value)) AS total_exposure,
    SUM(unrealized_pnl) AS total_pnl,
    SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
        NULLIF(SUM(ABS(position_value)), 0) AS long_pct
FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '2 hours'
  AND ABS(position_value) > 100000
GROUP BY wallet_id
HAVING SUM(ABS(position_value)) > 1000000;

-- Market Summary (refresh every 1 minute)
CREATE MATERIALIZED VIEW mv_market_summary AS
SELECT
    token,
    last("markPx", "timestampUTC") AS price,
    last(funding, "timestampUTC") AS funding,
    last("openInterest", "timestampUTC") AS open_interest,
    last("dayNtlVlm", "timestampUTC") AS volume_24h
FROM perp_tokens
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
GROUP BY token;

-- Liquidation Summary (refresh every 1 minute)
CREATE MATERIALIZED VIEW mv_liquidation_summary AS
SELECT
    token,
    COUNT(*) AS liq_count_24h,
    SUM(px * sz) AS liq_volume_24h,
    COUNT(*) FILTER (WHERE side IN ('B', 'LONG')) AS long_liqs,
    COUNT(*) FILTER (WHERE side IN ('A', 'SHORT')) AS short_liqs
FROM hl_liquidations
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
GROUP BY token;
```

---

## 8. Roadmap

### Phase 1: Foundation (Weeks 1-4)

| Week | Deliverable |
|------|-------------|
| 1 | Database views, API scaffolding |
| 2 | Core API endpoints (markets, whales) |
| 3 | Basic web dashboard (market overview) |
| 4 | Telegram bot (basic commands) |

**MVP Features:**
- Market overview page
- Whale tracker (top 50)
- Basic liquidation feed
- Telegram: `/whale`, `/token`, `/liq`

### Phase 2: Intelligence (Weeks 5-8)

| Week | Deliverable |
|------|-------------|
| 5 | Whale profiles, position tracking |
| 6 | Liquidation cascade detection |
| 7 | Alert system (Telegram) |
| 8 | Leaderboard + trader profiles |

**Features:**
- Whale detail pages
- Cascade alerts
- Follow whale feature
- Top trader rankings

### Phase 3: Advanced (Weeks 9-12)

| Week | Deliverable |
|------|-------------|
| 9 | Capital flows dashboard |
| 10 | Risk analytics page |
| 11 | Copy trading detection |
| 12 | Cross-exchange analysis |

**Features:**
- Flow monitoring
- Risk metrics display
- Correlated wallet detection
- HL vs Binance/Coinbase spreads

### Phase 4: Scale (Weeks 13-16)

| Week | Deliverable |
|------|-------------|
| 13 | REST API public launch |
| 14 | Webhook system |
| 15 | Performance optimization |
| 16 | Mobile-responsive design |

---

## 9. Competitive Position

### 9.1 Market Landscape

| Competitor | Focus | Weakness |
|------------|-------|----------|
| Coinglass | General perps data | No whale tracking |
| Hypurrscan | HL explorer | No analytics |
| Velo Data | HL data | Limited features |
| Laevitas | Options focus | Not HL-specific |

### 9.2 BlockLiquidity Advantages

| Advantage | Description |
|-----------|-------------|
| **Whale Intelligence** | Real-time whale tracking unavailable elsewhere |
| **Liquidation Cascades** | Predictive cascade detection |
| **Copy Signals** | Identify who to follow |
| **Cross-Exchange** | HL vs CEX comparison |
| **Multi-Channel** | Web + Telegram + API |
| **Data Depth** | 15B+ records historical |

---

## 10. Business Model

### 10.1 Revenue Streams

| Tier | Price | Features |
|------|-------|----------|
| **Free** | $0 | Basic dashboard, limited API (100 req/day) |
| **Pro** | $49/mo | Full dashboard, alerts, API (10K req/day) |
| **Whale** | $199/mo | Real-time WebSocket, webhooks, API (100K req/day) |
| **Enterprise** | Custom | Dedicated support, custom features |

### 10.2 Growth Metrics

| Metric | Target (6 months) |
|--------|-------------------|
| Registered Users | 10,000 |
| Daily Active Users | 1,500 |
| Pro Subscribers | 500 |
| API Users | 200 |
| MRR | $35,000 |

---

## Summary

BlockLiquidity leverages your existing 15B+ record Hyperliquid database to build a comprehensive whale intelligence and analytics platform. With real-time whale tracking, liquidation monitoring, and copy trading signals, it fills a gap in the market that no existing tool addresses.

**What You Have:**
- ✅ Complete historical data
- ✅ Real-time data pipeline
- ✅ All metrics queries ready

**What To Build:**
- API layer
- Web dashboard
- Telegram bot
- Alert system

**Time to MVP:** 4 weeks
**Time to Full Product:** 12-16 weeks

---

*Document Version: 1.0*
*Last Updated: January 2026*
