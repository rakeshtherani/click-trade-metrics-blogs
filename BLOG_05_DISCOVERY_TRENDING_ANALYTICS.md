# Building Real-Time Solana DEX Analytics with ClickHouse: Part 5

## Discovery Token Lifecycle, Trending Analytics & Materialized Views

*Part 5 of a 6-part series on building production-grade analytics for Solana DEX trading*

---

## Series Overview

| Part | Topic | Status |
|------|-------|--------|
| 1 | [Architecture Overview](./BLOG_01_ARCHITECTURE_OVERVIEW.md) | Published |
| 2 | [Trade Enrichment & OHLCV Aggregation](./BLOG_02_TRADE_ENRICHMENT_OHLCV.md) | Published |
| 3 | [Token Metrics & Rolling Windows](./BLOG_03_TOKEN_METRICS_ROLLING_WINDOWS.md) | Published |
| 4 | [Position Tracking & PnL Calculation](./BLOG_04_POSITION_TRACKING_PNL.md) | Published |
| 5 | **Discovery, Trending & Materialized Views** | **You are here** |
| 6 | Production Patterns & Query Optimization | Coming Soon |

---

## Introduction

In Parts 1-4, we built the foundational layers: trade enrichment, token metrics, and position tracking. Now we tackle two critical user-facing features:

1. **Discovery Page** - Real-time tracking of token lifecycle from launch to graduation
2. **Trending Analytics** - Aggregated metrics for market-wide analysis

These features introduce new ClickHouse patterns:
- **Multi-stage event tracking** with state transitions
- **AggregatingMergeTree** for pre-computed rollups
- **Materialized Views** for automatic aggregation
- **Scheduled jobs** for derived metrics

By the end of this article, you'll understand how to build analytics that can answer questions like:
- "Which tokens launched on PumpFun are about to graduate?"
- "What's the hourly trading intensity pattern?"
- "Who are the top-performing whale traders this week?"

---

## Table of Contents

1. [Discovery Token Architecture](#1-discovery-token-architecture)
2. [Data Pipeline: Discovery Events](#2-data-pipeline-discovery-events)
3. [The Token Lifecycle: Three Stages](#3-the-token-lifecycle-three-stages)
4. [Trending Analytics Architecture](#4-trending-analytics-architecture)
5. [AggregatingMergeTree Deep Dive](#5-aggregatingmergetree-deep-dive)
6. [Trending Widgets Implementation](#6-trending-widgets-implementation)
7. [Whale Metrics & Leaderboards](#7-whale-metrics--leaderboards)
8. [Scheduled Jobs & Maintenance](#8-scheduled-jobs--maintenance)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Discovery Token Architecture

### What is Token Discovery?

Token discovery tracks new tokens from their creation through their lifecycle:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TOKEN LIFECYCLE STAGES                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  NEWLY_CREATED   │───>│ ABOUT_TO_GRADUATE│───>│    GRADUATED     │  │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘  │
│         │                        │                       │              │
│         ▼                        ▼                       ▼              │
│  • Token minted           • Bonding curve       • Migrated to DEX      │
│  • Pool created             ~90% filled         • Liquidity added      │
│  • Bonding curve          • High momentum       • Full trading         │
│    0-50% filled           • About to migrate      enabled              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why Track Discovery?

For a trading platform, discovery is critical:

1. **Early Entry** - Traders want to find tokens before they explode
2. **Risk Detection** - Identify rug pulls via holder concentration
3. **Market Intelligence** - Track launch rates by platform
4. **Alpha Generation** - Spot patterns in successful graduations

### The Data Model

Each discovery token carries rich metadata:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DISCOVERY TOKEN FIELDS                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  IDENTITY              MARKET STATS           HOLDER ANALYSIS           │
│  ────────              ────────────           ───────────────           │
│  • token_mint          • price_sol/usd        • holders_count           │
│  • token_name          • market_cap_sol/usd   • top10_holdings_pct      │
│  • token_symbol        • liquidity_sol/usd    • dev_holdings_pct        │
│  • creator             • volume_24h_sol/usd   • snipers_pct             │
│  • created_at          • trade_count_24h      • bundler_pct             │
│                                               • insiders_pct            │
│                                                                         │
│  GRADUATION            SOCIAL SIGNALS         RISK INDICATORS           │
│  ──────────            ──────────────         ───────────────           │
│  • stage               • twitter              • twitter_reuse_count     │
│  • bonding_curve_pct   • telegram             • dev_funding_source      │
│  • protocols[]         • website              • concentration_risk      │
│  • primary_dex         • discord                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Pipeline: Discovery Events

### Overview

Discovery events flow through a multi-stage pipeline:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISCOVERY EVENT PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SOLANA                    DATASTREAMS                   CLICKHOUSE     │
│  INDEXER                   (Rustflow)                    INGESTION      │
│  ──────────────────────    ───────────────────────────   ─────────────  │
│                                                                         │
│  ┌──────────────┐          ┌────────────────────────┐                   │
│  │ Token Create │          │  discovery_processor   │                   │
│  │    Event     │─────────>│                        │                   │
│  └──────────────┘          │  1. Load token meta    │                   │
│                            │  2. Load holder data   │                   │
│  ┌──────────────┐          │  3. Compute stage      │                   │
│  │  Pool State  │─────────>│  4. Detect risks       │    ┌───────────┐ │
│  │   Updates    │          │  5. Emit to stage      │───>│ discovery │ │
│  └──────────────┘          │     subject            │    │  _queue   │ │
│                            └────────────────────────┘    └─────┬─────┘ │
│  ┌──────────────┐                                              │       │
│  │ Trade Events │─────────────────────────────────────────────>│       │
│  │  (activity)  │                                              │       │
│  └──────────────┘                                              ▼       │
│                                                          ┌───────────┐ │
│                            STAGE-BASED ROUTING           │ discovery │ │
│                            ────────────────────          │  (table)  │ │
│                            • newly_created ─────────────>└───────────┘ │
│                            • about_to_graduate ─────────>      │       │
│                            • graduated ─────────────────>      ▼       │
│                                                          ┌───────────┐ │
│                                                          │ discovery │ │
│                                                          │ _history  │ │
│                                                          └───────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Discovery Processor (Rustflow)

The discovery processor maintains state across multiple inputs:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     DISCOVERY PROCESSOR STATE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  INPUT EVENTS:                     STATE MAINTAINED:                    │
│                                                                         │
│  solana.tokens ─────────┐          ┌─────────────────────────────────┐ │
│                         │          │ TokenDiscoveryState             │ │
│  solana.pools ──────────┤          ├─────────────────────────────────┤ │
│                         ├─────────>│ • token_mint: String            │ │
│  solana.trades.v2 ──────┤          │ • current_stage: Stage          │ │
│                         │          │ • bonding_curve_pct: Decimal    │ │
│  enriched_trades ───────┘          │ • holder_snapshot: HolderData   │ │
│                                    │ • risk_flags: Vec<RiskFlag>     │ │
│                                    │ • last_trade_time: DateTime     │ │
│                                    │ • metrics_24h: RollingMetrics   │ │
│                                    └─────────────────────────────────┘ │
│                                                                         │
│  OUTPUT EVENTS (by stage):                                              │
│                                                                         │
│  solana.discovery.newly_created ────────> New tokens (< 50% bonding)   │
│  solana.discovery.about_to_graduate ────> Near graduation (> 90%)      │
│  solana.discovery.graduated ────────────> Migrated to DEX              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Stage Transition Logic

```
STAGE TRANSITION RULES:
═══════════════════════════════════════════════════════════════════════════

NEWLY_CREATED → ABOUT_TO_GRADUATE:
  IF bonding_curve_pct >= 90% THEN transition

ABOUT_TO_GRADUATE → GRADUATED:
  IF token migrated to DEX (Raydium, Orca, etc.) THEN transition

SPECIAL CASES:
  • Some tokens skip ABOUT_TO_GRADUATE (fast graduation)
  • Some tokens never graduate (abandoned/rugged)
  • Stage can ONLY move forward, never backward
```

### NATS Subject Architecture

Discovery uses **stage-based routing** for efficient consumption:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NATS SUBJECT HIERARCHY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  solana.discovery.*                                                     │
│  ├── solana.discovery.newly_created      High volume (100s/minute)     │
│  ├── solana.discovery.about_to_graduate  Medium volume (10s/minute)    │
│  └── solana.discovery.graduated          Low volume (1s/minute)        │
│                                                                         │
│  CONSUMER PATTERN:                                                      │
│  ═════════════════                                                      │
│  • Discovery Page API → Subscribes to all 3 subjects                   │
│  • Alert System → Subscribes only to about_to_graduate                 │
│  • Analytics → Subscribes to graduated for success metrics             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### RowBinary Serialization

Each stage emits the same DiscoveryToken structure in RowBinary format:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISCOVERYTOKEN STRUCT (28+ fields)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  FIELD LAYOUT (RowBinary order):                                        │
│  ────────────────────────────────                                       │
│                                                                         │
│  Offset   Type              Field                                       │
│  ──────   ────              ─────                                       │
│  0        String            token_mint                                  │
│  +N       String            token_symbol                                │
│  +N       String            token_name                                  │
│  +N       String            creator                                     │
│  +N       Enum8             stage (1=newly_created, 2=about_to, 3=grad) │
│  +N       Decimal(38,18)    bonding_curve_pct (nullable)                │
│  +N       Decimal(38,18)    top10_holdings_pct                          │
│  +N       Decimal(38,18)    snipers_pct                                 │
│  +N       UInt64            holders_count                               │
│  +N       Decimal(38,18)    bundler_pct                                 │
│  +N       Decimal(38,18)    insiders_pct                                │
│  +N       Decimal(38,18)    dev_holdings_pct                            │
│  +N       String            twitter                                     │
│  +N       String            telegram                                    │
│  +N       String            website                                     │
│  +N       String            discord                                     │
│  +N       Decimal(38,18)    market_cap_sol                              │
│  +N       Decimal(38,18)    market_cap_usd                              │
│  +N       Decimal(38,18)    price_sol                                   │
│  +N       Decimal(38,18)    price_usd                                   │
│  +N       Decimal(38,18)    liquidity_sol                               │
│  +N       Decimal(38,18)    liquidity_usd                               │
│  +N       Decimal(38,18)    supply                                      │
│  +N       Decimal(38,18)    volume_24h_sol                              │
│  +N       Decimal(38,18)    volume_24h_usd                              │
│  +N       UInt64            trade_count_24h                             │
│  +N       DateTime64(9)     created_at                                  │
│  +N       DateTime64(9)     last_updated                                │
│                                                                         │
│  WIRE SIZE: ~400-600 bytes (varies with string lengths)                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Token Lifecycle: Three Stages

### Stage 1: Newly Created

Tokens enter the system when first created on a launchpad:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      NEWLY CREATED TOKENS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CHARACTERISTICS:                                                       │
│  • Bonding curve: 0-50% filled                                         │
│  • Liquidity: Low (launchpad-provided)                                 │
│  • Holders: Typically < 100                                            │
│  • Trading: Limited to launchpad DEX                                   │
│                                                                         │
│  RISK SIGNALS TO WATCH:                                                 │
│  ─────────────────────                                                  │
│  ┌────────────────────┬──────────────────────────────────────────────┐ │
│  │ Risk Factor        │ What It Means                                │ │
│  ├────────────────────┼──────────────────────────────────────────────┤ │
│  │ dev_holdings > 10% │ Creator holds significant supply             │ │
│  │ snipers_pct > 20%  │ Bots front-ran launch, may dump             │ │
│  │ bundler_pct > 10%  │ Same wallet split into many, coordinated    │ │
│  │ top10 > 50%        │ Concentrated ownership, manipulation risk   │ │
│  │ twitter_reuse > 1  │ Same social used by other tokens (scam)     │ │
│  └────────────────────┴──────────────────────────────────────────────┘ │
│                                                                         │
│  VOLUME: ~100-500 new tokens per hour on active days                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Stage 2: About to Graduate

Tokens approaching the graduation threshold:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ABOUT TO GRADUATE TOKENS                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CHARACTERISTICS:                                                       │
│  • Bonding curve: 90-100% filled                                       │
│  • Liquidity: Medium (bonding curve nearly full)                       │
│  • Holders: Typically 100-1000                                         │
│  • Trading: High activity as traders position for graduation           │
│                                                                         │
│  WHY THIS STAGE MATTERS:                                                │
│  ───────────────────────                                                │
│  • Price often spikes 2-5x on graduation                               │
│  • Smart money accumulates here                                         │
│  • Window of opportunity: minutes to hours                             │
│                                                                         │
│  GRADUATION TIMELINE:                                                   │
│  ────────────────────                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  90%────────95%────────99%────────100%                          │   │
│  │   │          │          │          │                            │   │
│  │   │          │          │          └── GRADUATED (migrated)     │   │
│  │   │          │          │                                       │   │
│  │   │          │          └── Migration queued                    │   │
│  │   │          │                                                  │   │
│  │   │          └── High alert: graduation imminent               │   │
│  │   │                                                             │   │
│  │   └── Enter ABOUT_TO_GRADUATE stage                            │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  VOLUME: ~10-50 tokens per hour                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Stage 3: Graduated

Tokens that have migrated to a full DEX:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       GRADUATED TOKENS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CHARACTERISTICS:                                                       │
│  • Bonding curve: NULL (no longer applicable)                          │
│  • Liquidity: Added to Raydium/Orca/Meteora                            │
│  • Holders: Growing (exposed to wider market)                          │
│  • Trading: Full DEX functionality                                     │
│                                                                         │
│  GRADUATION OUTCOMES:                                                   │
│  ───────────────────                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     GRADUATED TOKEN FATES                       │   │
│  ├──────────────────┬──────────────────────────────────────────────┤   │
│  │  ~1-5%           │  Moon (10x+)                                 │   │
│  │  ~10-20%         │  Moderate success (2-5x)                     │   │
│  │  ~30-40%         │  Flat (maintains around graduation price)   │   │
│  │  ~40-50%         │  Decline (dumps post-graduation)            │   │
│  │  ~5-10%          │  Rug (liquidity pulled)                     │   │
│  └──────────────────┴──────────────────────────────────────────────┘   │
│                                                                         │
│  KEY METRICS AT GRADUATION:                                             │
│  ──────────────────────────                                             │
│  • avg_market_cap_usd: Typical graduation market cap                   │
│  • avg_hours_to_graduate: Time from creation to graduation             │
│  • avg_holders_at_graduation: Holder count when graduated              │
│                                                                         │
│  VOLUME: ~1-10 tokens per hour                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Trending Analytics Architecture

### The Dashboard Requirement

A trending page needs multiple aggregated views:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TRENDING PAGE WIDGETS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────┐  ┌─────────────────────────┐              │
│  │   LAUNCHPAD VOLUME      │  │  LAUNCHES & GRADUATIONS │              │
│  │   ─────────────────     │  │  ─────────────────────  │              │
│  │                         │  │                         │              │
│  │   PumpFun: $12.5M       │  │   PumpFun:              │              │
│  │   Moonshot: $3.2M       │  │     Launched: 1,234     │              │
│  │   Raydium: $45.1M       │  │     Graduated: 56       │              │
│  │   Orca: $8.7M           │  │     Grad Rate: 4.5%     │              │
│  │                         │  │                         │              │
│  └─────────────────────────┘  └─────────────────────────┘              │
│                                                                         │
│  ┌─────────────────────────┐  ┌─────────────────────────┐              │
│  │   TRADING INTENSITY     │  │    TRENDING TOKENS      │              │
│  │   ─────────────────     │  │    ───────────────      │              │
│  │                         │  │                         │              │
│  │   [Hourly Heatmap]      │  │   1. $PEPE  +234%       │              │
│  │   00 01 02 ... 23       │  │   2. $MOON  +156%       │              │
│  │   ▓▓ ░░ ░░     ▓▓       │  │   3. $DOGE  +89%        │              │
│  │   Peak: 14:00 UTC       │  │   4. $WIF   +67%        │              │
│  │                         │  │                         │              │
│  └─────────────────────────┘  └─────────────────────────┘              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      WHALE LEADERBOARD                          │   │
│  │                      ─────────────────                          │   │
│  │                                                                 │   │
│  │   Rank  Wallet      PnL         Win Rate   Trades   Tier       │   │
│  │   ────  ──────      ───         ────────   ──────   ────       │   │
│  │   1     ABC...123   +$1.2M      72%        1,234    MEGA_WHALE │   │
│  │   2     DEF...456   +$890K      68%        892      WHALE      │   │
│  │   3     GHI...789   +$456K      71%        2,341    WHALE      │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Aggregation Architecture

Raw events flow into pre-aggregated tables:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRENDING AGGREGATION FLOW                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│           SOURCE TABLES                    AGGREGATION TABLES           │
│           ─────────────                    ──────────────────           │
│                                                                         │
│  ┌─────────────────┐                  ┌──────────────────────────┐     │
│  │ enriched_trades │──────────────────>│ trending_launchpad_      │     │
│  │                 │      │           │ volume_daily             │     │
│  └─────────────────┘      │           └──────────────────────────┘     │
│                           │                                             │
│                           │           ┌──────────────────────────┐     │
│                           └──────────>│ trending_hourly_         │     │
│                                       │ intensity                │     │
│  ┌─────────────────┐                  └──────────────────────────┘     │
│  │ discovery       │                                                    │
│  │                 │──────────────────>┌──────────────────────────┐    │
│  │                 │     │            │ trending_launches_daily  │     │
│  └─────────────────┘     │            └──────────────────────────┘     │
│                          │                                              │
│                          └───────────>┌──────────────────────────┐     │
│                                       │ trending_graduations_    │     │
│  ┌─────────────────┐                  │ daily                    │     │
│  │ token_metrics   │                  └──────────────────────────┘     │
│  │                 │──────────────────>┌──────────────────────────┐    │
│  │                 │                  │ trending_token_holder_   │     │
│  └─────────────────┘                  │ snapshots                │     │
│                                       └──────────────────────────┘     │
│  ┌─────────────────┐                                                    │
│  │ position_       │──────────────────>┌──────────────────────────┐    │
│  │ overview        │                  │ trending_whale_stats     │     │
│  │                 │                  │                          │     │
│  └─────────────────┘                  └──────────────────────────┘     │
│                                                                         │
│                                               │                         │
│                                               ▼                         │
│                                       ┌──────────────────────────┐     │
│                                       │       API VIEWS          │     │
│                                       │  v_launchpad_volume      │     │
│                                       │  v_launches_and_grads    │     │
│                                       │  v_trading_intensity     │     │
│                                       │  v_trending_tokens       │     │
│                                       │  v_whale_leaderboard     │     │
│                                       └──────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. AggregatingMergeTree Deep Dive

### Why AggregatingMergeTree?

For trending analytics, we need pre-computed aggregations:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   THE AGGREGATION PROBLEM                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  NAIVE APPROACH (Query-time aggregation):                               │
│  ─────────────────────────────────────────                              │
│                                                                         │
│  SELECT                                                                 │
│      dex AS venue,                                                      │
│      sum(volume_usd) AS volume_24h                                      │
│  FROM enriched_trades                                                   │
│  WHERE block_timestamp >= now() - INTERVAL 24 HOUR                      │
│  GROUP BY venue                                                         │
│                                                                         │
│  PROBLEM: Scans 10-50M rows per query                                  │
│  LATENCY: 2-10 seconds (unacceptable for dashboard)                    │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  AGGREGATINGMERGETREE APPROACH:                                         │
│  ────────────────────────────────                                       │
│                                                                         │
│  1. Materialize daily aggregations as rows arrive                       │
│  2. Store aggregate states (not raw values)                             │
│  3. Query pre-computed aggregates                                       │
│                                                                         │
│  RESULT: Scans 30 rows (1 row per day)                                 │
│  LATENCY: 5-20 milliseconds                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### How Aggregate States Work

ClickHouse stores intermediate aggregate states, not final values:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGGREGATE STATE STORAGE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  REGULAR COLUMN:                                                        │
│  ───────────────                                                        │
│  volume_usd Decimal(38, 18)  → Stores: 1234.56                         │
│                                                                         │
│  SIMPLE AGGREGATE FUNCTION:                                             │
│  ──────────────────────────                                             │
│  volume_usd SimpleAggregateFunction(sum, Decimal128(18))               │
│  → Stores: 1234.56 (same, but knows it's a partial sum)                │
│  → On merge: Adds values together                                       │
│                                                                         │
│  AGGREGATE FUNCTION (complex):                                          │
│  ─────────────────────────────                                          │
│  unique_traders AggregateFunction(uniq, String)                         │
│  → Stores: HyperLogLog sketch (binary blob)                             │
│  → On merge: Combines sketches                                          │
│  → On query: uniqMerge(unique_traders) → 12345                         │
│                                                                         │
│  MERGE BEHAVIOR:                                                        │
│  ───────────────                                                        │
│                                                                         │
│  ┌────────────┐     ┌────────────┐                                     │
│  │ Part 1     │     │ Part 2     │                                     │
│  │ date: 1/15 │     │ date: 1/15 │                                     │
│  │ venue: pf  │  +  │ venue: pf  │                                     │
│  │ vol: 1000  │     │ vol: 2000  │                                     │
│  │ uniq: [HLL]│     │ uniq: [HLL]│                                     │
│  └────────────┘     └────────────┘                                     │
│         │                 │                                             │
│         └────────┬────────┘                                             │
│                  ▼                                                       │
│         ┌────────────────┐                                              │
│         │ Merged Part    │                                              │
│         │ date: 1/15     │                                              │
│         │ venue: pf      │                                              │
│         │ vol: 3000      │  (sum of 1000 + 2000)                        │
│         │ uniq: [merged] │  (HLL sketches combined)                     │
│         └────────────────┘                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Table Design Pattern

```sql
-- Pattern: AggregatingMergeTree for daily aggregations

CREATE TABLE datastreams.trending_launchpad_volume_daily
(
    -- DIMENSIONS (partition/order keys)
    date Date,
    venue LowCardinality(String),

    -- SIMPLE AGGREGATES (additive - can sum across parts)
    volume_sol SimpleAggregateFunction(sum, Decimal128(18)),
    volume_usd SimpleAggregateFunction(sum, Decimal128(18)),
    trade_count SimpleAggregateFunction(sum, UInt64),
    buy_count SimpleAggregateFunction(sum, UInt64),
    sell_count SimpleAggregateFunction(sum, UInt64),

    -- COMPLEX AGGREGATES (non-additive - need special handling)
    unique_tokens AggregateFunction(uniq, String),
    unique_traders AggregateFunction(uniq, String),

    -- AVERAGE (needs count for proper merge)
    avg_trade_size_sol SimpleAggregateFunction(avg, Decimal128(18))
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, venue)
TTL date + INTERVAL 365 DAY;
```

### Materialized View Population

```sql
-- MV that populates the aggregation table

CREATE MATERIALIZED VIEW datastreams.trending_launchpad_volume_daily_mv
TO datastreams.trending_launchpad_volume_daily AS
SELECT
    toDate(block_timestamp) AS date,
    dex AS venue,

    -- Simple aggregates
    sum(volume_sol) AS volume_sol,
    sum(volume_usd) AS volume_usd,
    count() AS trade_count,
    countIf(trade_direction = 0) AS buy_count,
    countIf(trade_direction = 1) AS sell_count,

    -- Complex aggregates (use *State functions)
    uniqState(token_mint) AS unique_tokens,
    uniqState(trader) AS unique_traders,

    avg(volume_sol) AS avg_trade_size_sol

FROM datastreams.enriched_trades
GROUP BY date, venue;
```

### Querying Aggregate Tables

```sql
-- View that provides the API layer

CREATE VIEW datastreams.v_launchpad_volume AS
SELECT
    venue,

    -- 24h metrics (simple: just sum)
    sumIf(volume_sol, date = today()) AS volume_24h_sol,
    sumIf(trade_count, date = today()) AS trades_24h,

    -- 24h unique counts (complex: use *Merge functions)
    uniqMergeIf(unique_tokens, date = today()) AS tokens_24h,
    uniqMergeIf(unique_traders, date = today()) AS traders_24h,

    -- 7d metrics
    sumIf(volume_usd, date >= today() - 6) AS volume_7d_usd,
    uniqMergeIf(unique_traders, date >= today() - 6) AS traders_7d,

    -- 30d metrics
    sumIf(volume_usd, date >= today() - 29) AS volume_30d_usd,

    -- Change calculation
    sumIf(volume_usd, date = today())
      - sumIf(volume_usd, date = today() - 1) AS volume_change_vs_yesterday

FROM datastreams.trending_launchpad_volume_daily
WHERE date >= today() - 29
GROUP BY venue
ORDER BY volume_24h_usd DESC;
```

### Performance Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE COMPARISON                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  QUERY: "Volume by venue for last 30 days"                             │
│                                                                         │
│  WITHOUT AGGREGATION (raw scan):                                        │
│  ────────────────────────────────                                       │
│  • Rows scanned: 150,000,000                                           │
│  • Data read: 12.5 GB                                                  │
│  • Query time: 8.2 seconds                                             │
│  • CPU usage: 100% across 8 cores                                      │
│                                                                         │
│  WITH AGGREGATINGMERGETREE:                                             │
│  ──────────────────────────                                             │
│  • Rows scanned: 150 (30 days × 5 venues)                              │
│  • Data read: 12 KB                                                    │
│  • Query time: 8 milliseconds                                          │
│  • CPU usage: Single core, minimal                                     │
│                                                                         │
│  SPEEDUP: 1000x faster                                                 │
│  DATA REDUCTION: 1,000,000x less data read                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Trending Widgets Implementation

### Widget 1: Launches & Graduations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   LAUNCHES & GRADUATIONS PIPELINE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  DATA FLOW:                                                             │
│                                                                         │
│  discovery (stage=newly_created) ───> trending_launches_daily          │
│  discovery (stage=graduated) ───────> trending_graduations_daily       │
│                                              │                          │
│                                              ▼                          │
│                                    v_launches_and_graduations           │
│                                                                         │
│  AGGREGATION TABLE (Launches):                                          │
│  ─────────────────────────────                                          │
│                                                                         │
│  CREATE TABLE trending_launches_daily (                                 │
│      date Date,                                                         │
│      venue LowCardinality(String),                                      │
│      launch_count SimpleAggregateFunction(sum, UInt64),                 │
│      launches_with_twitter SimpleAggregateFunction(sum, UInt64),        │
│      launches_with_telegram SimpleAggregateFunction(sum, UInt64),       │
│      launches_with_any_social SimpleAggregateFunction(sum, UInt64),     │
│      total_initial_liquidity_sol SimpleAggregateFunction(sum, ...),     │
│  ) ENGINE = AggregatingMergeTree()                                      │
│    ORDER BY (date, venue);                                              │
│                                                                         │
│  COMBINED VIEW OUTPUT:                                                  │
│  ─────────────────────                                                  │
│  ┌───────────┬──────────┬──────────┬────────────┬───────────────────┐  │
│  │ venue     │ 24h      │ 7d       │ grad_24h   │ grad_ratio_7d     │  │
│  │           │ launches │ launches │            │                   │  │
│  ├───────────┼──────────┼──────────┼────────────┼───────────────────┤  │
│  │ pumpfun   │ 1,234    │ 8,567    │ 56         │ 4.5%              │  │
│  │ moonshot  │ 234      │ 1,678    │ 12         │ 3.2%              │  │
│  │ raydium   │ 45       │ 312      │ 8          │ 8.9%              │  │
│  └───────────┴──────────┴──────────┴────────────┴───────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Widget 2: Trading Intensity Heatmap

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   TRADING INTENSITY HEATMAP                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PURPOSE: Show hourly trading patterns vs 7-day rolling average         │
│                                                                         │
│  DATA FLOW:                                                             │
│                                                                         │
│  enriched_trades ───> trending_hourly_intensity ───> v_trading_intensity│
│                                                                         │
│  AGGREGATION TABLE:                                                     │
│  ──────────────────                                                     │
│                                                                         │
│  CREATE TABLE trending_hourly_intensity (                               │
│      date Date,                                                         │
│      hour UInt8,  -- 0-23 UTC                                          │
│      volume_sol SimpleAggregateFunction(sum, Decimal128(18)),           │
│      trade_count SimpleAggregateFunction(sum, UInt64),                  │
│      unique_traders AggregateFunction(uniq, String)                     │
│  ) ENGINE = AggregatingMergeTree()                                      │
│    ORDER BY (date, hour)                                                │
│    TTL date + INTERVAL 90 DAY;                                          │
│                                                                         │
│  VIEW WITH INTENSITY SCORE:                                             │
│  ──────────────────────────                                             │
│                                                                         │
│  intensity_score = CASE                                                 │
│    WHEN volume > 2x avg   THEN 100 (Hot)                               │
│    WHEN volume > 1.5x avg THEN 80                                      │
│    WHEN volume > avg      THEN 60                                      │
│    WHEN volume > 0.5x avg THEN 40                                      │
│    ELSE 20 (Cold)                                                       │
│  END                                                                    │
│                                                                         │
│  HEATMAP OUTPUT:                                                        │
│  ───────────────                                                        │
│                                                                         │
│  Hour:  00  01  02  03  04  05  06  07  08  09  10  11  12  ...        │
│  Score: 20  20  20  20  40  60  80  100 100 80  60  60  80  ...        │
│         ░░  ░░  ░░  ░░  ▒▒  ▓▓  ██  ██  ██  ██  ▓▓  ▓▓  ██             │
│                                                                         │
│         ^                 ^                   ^                         │
│         |                 |                   |                         │
│         Low (Asia sleep)  US morning          US lunch                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Widget 3: Trending Tokens

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRENDING TOKENS CALCULATION                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRENDING SCORE FORMULA:                                                │
│  ───────────────────────                                                │
│                                                                         │
│  trending_score = (                                                     │
│      price_momentum_score * 0.40      -- 40% weight                     │
│    + volume_score * 0.30              -- 30% weight                     │
│    + holder_growth_score * 0.30       -- 30% weight                     │
│  )                                                                      │
│                                                                         │
│  COMPONENT CALCULATIONS:                                                │
│  ───────────────────────                                                │
│                                                                         │
│  price_momentum_score:                                                  │
│    = min(price_change_1h_pct, 100)   -- Cap at 100% gain               │
│                                                                         │
│  volume_score:                                                          │
│    = min(volume_24h_usd / 100000 * 30, 30)                              │
│    -- $100K volume = full 30 points                                     │
│                                                                         │
│  holder_growth_score:                                                   │
│    = min((current_holders - holders_1h_ago) / holders_1h_ago * 100, 30) │
│    -- 100% holder growth = full 30 points                               │
│                                                                         │
│  DATA SOURCES:                                                          │
│  ─────────────                                                          │
│                                                                         │
│  token_metrics (FINAL) ─────┬───> Current price, volume, holders        │
│                             │                                           │
│  trending_token_holder_ ────┴───> Historical holder counts              │
│  snapshots (hourly)                                                     │
│                                                                         │
│  SNAPSHOT TABLE:                                                        │
│  ───────────────                                                        │
│                                                                         │
│  CREATE TABLE trending_token_holder_snapshots (                         │
│      token_mint String,                                                 │
│      snapshot_time DateTime,  -- Hourly snapshots                       │
│      holders_count UInt64,                                              │
│      price_usd Decimal(38, 18),                                         │
│      market_cap_usd Decimal(38, 18)                                     │
│  ) ENGINE = MergeTree()                                                 │
│    ORDER BY (token_mint, snapshot_time)                                 │
│    TTL snapshot_time + INTERVAL 7 DAY;  -- Only need 7 days            │
│                                                                         │
│  HOLDER GROWTH CALCULATION:                                             │
│  ──────────────────────────                                             │
│                                                                         │
│  WITH holder_growth AS (                                                │
│      SELECT                                                             │
│          token_mint,                                                    │
│          argMax(holders_count, snapshot_time) AS current,               │
│          argMaxIf(holders_count, snapshot_time,                         │
│              snapshot_time <= now() - INTERVAL 1 HOUR) AS holders_1h    │
│      FROM trending_token_holder_snapshots                               │
│      WHERE snapshot_time >= now() - INTERVAL 2 HOUR                     │
│      GROUP BY token_mint                                                │
│  )                                                                      │
│  SELECT                                                                 │
│      *,                                                                 │
│      (current - holders_1h) / holders_1h * 100 AS growth_1h_pct         │
│  FROM holder_growth                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Whale Metrics & Leaderboards

### Whale Stats Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHALE STATS PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SOURCE: position_overview (per wallet-token PnL)                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                position_overview (per position)                  │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  trader: ABC...123                                              │   │
│  │  token_mint: TOKEN1        realized_pnl_sol: +5.2               │   │
│  │                                                                 │   │
│  │  trader: ABC...123                                              │   │
│  │  token_mint: TOKEN2        realized_pnl_sol: -1.3               │   │
│  │                                                                 │   │
│  │  trader: ABC...123                                              │   │
│  │  token_mint: TOKEN3        realized_pnl_sol: +12.8              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│                    │                                                    │
│                    │ SCHEDULED JOB (every 5 min)                        │
│                    │ Aggregates by trader                               │
│                    ▼                                                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                 trending_whale_stats (per wallet)                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  trader: ABC...123                                              │   │
│  │  total_trades: 3                                                │   │
│  │  winning_trades: 2                                              │   │
│  │  losing_trades: 1                                               │   │
│  │  realized_pnl_sol: +16.7  (sum across all positions)            │   │
│  │  unique_tokens_traded: 3                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│                    │                                                    │
│                    │ VIEW with computed metrics                         │
│                    ▼                                                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                 v_whale_leaderboard                              │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  trader: ABC...123                                              │   │
│  │  win_rate_pct: 66.7%  (2/3 trades)                              │   │
│  │  realized_pnl_sol: +16.7                                        │   │
│  │  avg_pnl_per_trade: +5.57                                       │   │
│  │  roi_pct: 42.3%                                                 │   │
│  │  tier: WHALE  (based on total PnL)                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Whale Tier Classification

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      WHALE TIER SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TIER THRESHOLDS (by realized PnL USD):                                 │
│  ──────────────────────────────────────                                 │
│                                                                         │
│  ┌────────────────┬─────────────────┬────────────────────────────────┐ │
│  │ Tier           │ PnL Threshold   │ Description                    │ │
│  ├────────────────┼─────────────────┼────────────────────────────────┤ │
│  │ MEGA_WHALE     │ >= $1,000,000   │ Top 0.01% of traders           │ │
│  │ WHALE          │ >= $100,000     │ Top 0.1% of traders            │ │
│  │ DOLPHIN        │ >= $10,000      │ Top 1% of traders              │ │
│  │ FISH           │ >= $1,000       │ Top 10% of traders             │ │
│  │ SHRIMP         │ < $1,000        │ Most traders                   │ │
│  └────────────────┴─────────────────┴────────────────────────────────┘ │
│                                                                         │
│  ADDITIONAL METRICS:                                                    │
│  ───────────────────                                                    │
│                                                                         │
│  profit_factor = winning_trades / losing_trades                         │
│    • > 2.0 = Excellent                                                 │
│    • > 1.5 = Good                                                      │
│    • > 1.0 = Break-even                                                │
│    • < 1.0 = Losing                                                    │
│                                                                         │
│  avg_pnl_per_trade = realized_pnl / total_trades                        │
│    • Measures consistency                                              │
│                                                                         │
│  roi_pct = realized_pnl / (total_volume / 2) * 100                      │
│    • Dividing by 2 estimates buy volume                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Table Design

```sql
CREATE TABLE datastreams.trending_whale_stats
(
    -- Wallet identification
    trader String,

    -- Trade counts
    total_trades UInt64,
    winning_trades UInt64,
    losing_trades UInt64,

    -- Volume
    total_volume_sol Decimal(38, 18),
    total_volume_usd Decimal(38, 18),

    -- PnL
    realized_pnl_sol Decimal(38, 18),
    realized_pnl_usd Decimal(38, 18),

    -- Diversity
    unique_tokens_traded UInt64,

    -- Activity tracking
    first_trade DateTime64(9),
    last_trade DateTime64(9),
    active_days UInt32,

    -- Version for ReplacingMergeTree deduplication
    version UInt64 MATERIALIZED toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/trending_whale_stats',
    '{replica}',
    version
)
ORDER BY (trader);
```

---

## 8. Scheduled Jobs & Maintenance

### Job Architecture

Some aggregations cannot use MVs alone:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEDULED JOB ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WHY NOT JUST MVs?                                                      │
│  ─────────────────                                                      │
│                                                                         │
│  CASE 1: Whale Stats                                                    │
│  • Need to aggregate ACROSS all tokens per wallet                       │
│  • MV triggers per-row, can't do cross-row aggregation                  │
│  • Solution: Scheduled job that scans position_overview                 │
│                                                                         │
│  CASE 2: Twitter Reuse Count                                            │
│  • Need to count tokens with same Twitter                               │
│  • Requires self-join on discovery table                                │
│  • Solution: Scheduled job that updates the count                       │
│                                                                         │
│  CASE 3: Holder Snapshots                                               │
│  • Need point-in-time snapshots (not event-driven)                      │
│  • Solution: Hourly job that captures current state                     │
│                                                                         │
│  JOB SCHEDULE:                                                          │
│  ─────────────                                                          │
│                                                                         │
│  ┌────────────────────────────┬────────────┬─────────────────────────┐ │
│  │ Job                        │ Frequency  │ Duration                │ │
│  ├────────────────────────────┼────────────┼─────────────────────────┤ │
│  │ refresh_whale_stats        │ 5 min      │ ~10-30 seconds          │ │
│  │ snapshot_holders           │ 1 hour     │ ~5-10 seconds           │ │
│  │ update_twitter_reuse       │ 5 min      │ ~2-5 seconds            │ │
│  │ update_protocols_array     │ 1 min      │ ~1-2 seconds            │ │
│  └────────────────────────────┴────────────┴─────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Job Implementation

**Job 1: Whale Stats Refresh**

```sql
-- Run every 5 minutes via cron or ClickHouse scheduled tasks

INSERT INTO datastreams.trending_whale_stats
SELECT
    trader,

    -- Count positions as trades
    count() AS total_trades,
    countIf(realized_pnl_sol > 0) AS winning_trades,
    countIf(realized_pnl_sol < 0) AS losing_trades,

    -- Sum volumes
    sum(total_spent_sol + total_received_sol) AS total_volume_sol,
    sum(total_spent_usd + total_received_usd) AS total_volume_usd,

    -- Sum PnL
    sum(realized_pnl_sol) AS realized_pnl_sol,
    sum(realized_pnl_usd) AS realized_pnl_usd,

    -- Count unique tokens
    uniq(token_mint) AS unique_tokens_traded,

    -- Activity bounds
    min(last_updated) AS first_trade,
    max(last_updated) AS last_trade,
    uniq(toDate(last_updated)) AS active_days

FROM datastreams.position_overview FINAL
WHERE total_tokens_bought > 0  -- Has traded
GROUP BY trader
HAVING sum(abs(realized_pnl_sol)) > 1;  -- Minimum activity
```

**Job 2: Holder Snapshots**

```sql
-- Run every hour at :05

INSERT INTO datastreams.trending_token_holder_snapshots
SELECT
    token_mint,
    toStartOfHour(now()) AS snapshot_time,
    holders_count,
    top10_holdings_pct,
    dev_holdings_pct,
    price_usd,
    market_cap_usd,
    liquidity_usd,
    stats_24h_volume_usd AS volume_24h_usd
FROM datastreams.token_metrics FINAL
WHERE stats_24h_volume_usd > 1000;  -- Only active tokens
```

### Cron Setup

```bash
# /etc/cron.d/clickhouse-jobs

# Whale stats - every 5 minutes
*/5 * * * * clickhouse-user /usr/bin/clickhouse-client \
    --host localhost \
    --queries-file /opt/datastreams/sql/jobs/refresh_whale_stats.sql \
    >> /var/log/clickhouse/whale_stats.log 2>&1

# Holder snapshots - every hour at :05
5 * * * * clickhouse-user /usr/bin/clickhouse-client \
    --host localhost \
    --queries-file /opt/datastreams/sql/jobs/snapshot_holders.sql \
    >> /var/log/clickhouse/holder_snapshots.log 2>&1

# Twitter reuse - every 5 minutes
*/5 * * * * clickhouse-user /usr/bin/clickhouse-client \
    --host localhost \
    --queries-file /opt/datastreams/sql/jobs/update_twitter_reuse.sql \
    >> /var/log/clickhouse/twitter_reuse.log 2>&1
```

### Monitoring Jobs

```sql
-- Check last successful job runs
SELECT
    query,
    event_time,
    query_duration_ms / 1000 AS duration_sec,
    result_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE 'INSERT INTO datastreams.trending_%'
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY event_time DESC
LIMIT 20;

-- Alert if job is stale
SELECT
    'whale_stats' AS job,
    max(last_trade) AS last_update,
    dateDiff('minute', max(last_trade), now()) AS staleness_minutes
FROM datastreams.trending_whale_stats FINAL
HAVING staleness_minutes > 10;  -- Alert threshold
```

---

## 9. Key Takeaways

### Discovery Token Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     KEY CONCEPTS: DISCOVERY                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. THREE STAGES: newly_created → about_to_graduate → graduated         │
│     • Stage determined by bonding curve % and migration status          │
│     • Stages only move forward, never backward                          │
│                                                                         │
│  2. STAGE-BASED ROUTING via NATS subjects                               │
│     • Each stage has its own subject                                    │
│     • Consumers subscribe to stages they care about                     │
│     • Enables different retention, alerting per stage                   │
│                                                                         │
│  3. DUAL TABLE PATTERN                                                  │
│     • discovery (ReplacingMergeTree): Current state                    │
│     • discovery_history (MergeTree): Time-series history               │
│                                                                         │
│  4. RISK DETECTION built into the struct                                │
│     • snipers_pct, bundler_pct, insiders_pct                           │
│     • twitter_reuse_count for scam detection                           │
│     • dev_holdings_pct for rug risk                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### AggregatingMergeTree Patterns

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 KEY CONCEPTS: AGGREGATIONS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. USE AggregatingMergeTree FOR:                                       │
│     • Daily/hourly rollups                                              │
│     • Multi-timeframe aggregations (24h, 7d, 30d)                       │
│     • High-cardinality unique counts                                    │
│                                                                         │
│  2. COLUMN TYPES:                                                       │
│     • SimpleAggregateFunction(sum, ...) - Additive metrics             │
│     • AggregateFunction(uniq, ...) - HyperLogLog for uniques           │
│                                                                         │
│  3. QUERY PATTERNS:                                                     │
│     • Simple: sumIf(volume, date = today())                            │
│     • Complex: uniqMergeIf(unique_traders, date >= today() - 6)        │
│                                                                         │
│  4. MV PATTERNS:                                                        │
│     • Source → MV → Aggregation table                                  │
│     • Use *State functions in MV                                       │
│     • Use *Merge functions in views                                    │
│                                                                         │
│  5. PERFORMANCE:                                                        │
│     • 1000x faster than raw table scans                                │
│     • Essential for dashboard queries                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scheduled Jobs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 KEY CONCEPTS: SCHEDULED JOBS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. WHEN TO USE JOBS vs MVs:                                            │
│     • MVs: Per-row transformations, single-source aggregations          │
│     • Jobs: Cross-row aggregations, self-joins, point-in-time snapshots │
│                                                                         │
│  2. JOB DESIGN:                                                         │
│     • Use INSERT ... SELECT (not UPDATE)                                │
│     • Target ReplacingMergeTree for deduplication                       │
│     • Include version column for merge                                  │
│                                                                         │
│  3. MONITORING:                                                         │
│     • Track staleness via last_updated timestamps                       │
│     • Alert if job runs > expected duration                             │
│     • Log to queryable table for audit                                  │
│                                                                         │
│  4. RELIABILITY:                                                        │
│     • Jobs are idempotent (safe to re-run)                              │
│     • Use cron + lockfiles to prevent overlap                           │
│     • ReplacingMergeTree handles duplicate inserts                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next

In **Part 6**, we'll cover production patterns:
- Query optimization techniques
- Monitoring and alerting
- Capacity planning
- Common pitfalls and solutions
- Performance tuning for high-traffic scenarios

---

## Quick Reference

### Tables Created in This Part

| Table | Engine | Purpose | TTL |
|-------|--------|---------|-----|
| `discovery` | ReplacingMergeTree | Current discovery token state | None |
| `discovery_history` | MergeTree | Discovery state history | 90 days |
| `trending_launchpad_volume_daily` | AggregatingMergeTree | Daily volume by venue | 365 days |
| `trending_launches_daily` | AggregatingMergeTree | Daily launch counts | 365 days |
| `trending_graduations_daily` | AggregatingMergeTree | Daily graduation counts | 365 days |
| `trending_hourly_intensity` | AggregatingMergeTree | Hourly trading volume | 90 days |
| `trending_token_holder_snapshots` | MergeTree | Hourly holder snapshots | 7 days |
| `trending_whale_stats` | ReplacingMergeTree | Aggregated wallet stats | None |

### Views Created

| View | Purpose |
|------|---------|
| `v_discovery` | User-friendly discovery fields |
| `v_launchpad_volume` | 24h/7d/30d volume rollups |
| `v_launches_and_graduations` | Combined launch/grad metrics |
| `v_trading_intensity_hourly` | Hourly heatmap with 7d avg |
| `v_trending_tokens` | Trending score calculation |
| `v_whale_leaderboard` | Wallet PnL and tier |

---

*Continue to [Part 6: Production Patterns & Query Optimization](./BLOG_06_PRODUCTION_PATTERNS.md)*
