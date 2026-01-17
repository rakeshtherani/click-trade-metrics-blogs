# Part 7: Hyperliquid Perpetual Futures Analytics in ClickHouse

*A comprehensive guide to building real-time perps trading analytics with position tracking, funding payments, liquidation monitoring, and portfolio management*

---

## Introduction

While spot DEX trading dominates Solana, perpetual futures have become the backbone of crypto derivatives trading. Hyperliquid, running on Arbitrum L2, offers a high-performance on-chain order book with sub-second settlement, transparent execution, and deep liquidity. In this part, we extend our analytics platform to capture the full spectrum of perps trading activity.

Perpetual futures analytics introduce fundamentally different challenges compared to spot trading:

| Aspect | Spot Trading | Perpetual Futures |
|--------|--------------|-------------------|
| **State Model** | Stateless swaps | Stateful positions |
| **P&L Calculation** | Buy/sell spread | Mark-to-market + realized |
| **Funding** | None | Every 8 hours |
| **Leverage** | None | Up to 50x |
| **Liquidation Risk** | None | Continuous monitoring |
| **Direction** | Long only (own asset) | Long or short |

The patterns we've established for Solana spot markets translate directly to Hyperliquid, but the data model requires careful consideration of **position lifecycle management**, **real-time risk monitoring**, and **cumulative P&L tracking**.

---

## Understanding Perpetual Futures Mechanics

Before diving into implementation, let's understand the key concepts that drive our data model.

### Position Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PERPETUAL POSITION LIFECYCLE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐    ┌─────────────────┐    ┌───────────────────────────┐  │
│   │   DEPOSIT   │───▶│   OPEN TRADE    │───▶│   POSITION ACTIVE         │  │
│   │  (margin)   │    │  (first fill)   │    │                           │  │
│   └─────────────┘    └─────────────────┘    │  • Mark-to-market PnL     │  │
│                                              │  • Funding payments       │  │
│                                              │  • Liquidation monitoring │  │
│                                              └───────────┬───────────────┘  │
│                                                          │                  │
│                              ┌────────────────────────────┼──────────────┐  │
│                              ▼                            ▼              ▼  │
│                      ┌─────────────┐            ┌─────────────┐  ┌───────┐ │
│                      │ CLOSE TRADE │            │ LIQUIDATION │  │ADD TO │ │
│                      │ (voluntary) │            │ (forced)    │  │POSITION│ │
│                      └──────┬──────┘            └──────┬──────┘  └───┬───┘ │
│                             │                          │             │      │
│                             ▼                          ▼             │      │
│                      ┌─────────────┐            ┌─────────────┐      │      │
│                      │REALIZE PnL  │            │ REALIZE     │◀─────┘      │
│                      │(profit/loss)│            │ LOSS        │             │
│                      └──────┬──────┘            └──────┬──────┘             │
│                             │                          │                    │
│                             ▼                          ▼                    │
│                      ┌─────────────────────────────────────────┐           │
│                      │              WITHDRAW                    │           │
│                      │         (remaining margin)               │           │
│                      └─────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Metrics We Track

1. **Trade Fills**: Individual order executions with price, size, side, and P&L
2. **Position State**: Current exposure with entry price, mark price, and unrealized P&L
3. **Funding Payments**: Periodic payments between longs and shorts (every 8 hours)
4. **Account Overview**: Portfolio-level aggregation of all positions and activity
5. **Raw L1 Events**: Deposits, withdrawals, and liquidations from Arbitrum

---

## Data Pipeline Architecture

### Complete Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        HYPERLIQUID PERPS DATA PIPELINE                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                          DATA SOURCES                                     │  │
│  │                                                                           │  │
│  │   ┌─────────────────────┐        ┌────────────────────────────────────┐  │  │
│  │   │  Hyperliquid L1     │        │  Hyperliquid WebSocket API         │  │  │
│  │   │  (Arbitrum Events)  │        │  (Real-time User Subscriptions)    │  │  │
│  │   │                     │        │                                     │  │  │
│  │   │  • Deposits         │        │  • Trade Fills                      │  │  │
│  │   │  • Withdrawals      │        │  • Funding Payments                 │  │  │
│  │   │  • Liquidations     │        │  • Position Updates                 │  │  │
│  │   └──────────┬──────────┘        └──────────────┬─────────────────────┘  │  │
│  └──────────────┼───────────────────────────────────┼───────────────────────┘  │
│                 │                                   │                          │
│                 ▼                                   ▼                          │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                        HYPERLIQUID INDEXER                                │  │
│  │                                                                           │  │
│  │   Rust service that:                                                      │  │
│  │   • Subscribes to Hyperliquid WebSocket per Click user                   │  │
│  │   • Indexes L1 events from Arbitrum                                       │  │
│  │   • Serializes events using Bincode                                       │  │
│  │   • Publishes to NATS JetStream                                          │  │
│  └──────────────────────────────────────┬───────────────────────────────────┘  │
│                                         │                                      │
│                                         ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                          NATS JETSTREAM                                   │  │
│  │                                                                           │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │  │
│  │   │                    RAW EVENT SUBJECTS                            │    │  │
│  │   │                                                                  │    │  │
│  │   │  hyperliquid.fills ─────────────► Trade executions              │    │  │
│  │   │  hyperliquid.funding ───────────► 8-hour funding payments       │    │  │
│  │   │  hyperliquid.deposits ──────────► L1 deposits to exchange       │    │  │
│  │   │  hyperliquid.withdrawals ───────► Withdrawals to L1             │    │  │
│  │   │  hyperliquid.liquidations ──────► Forced position closures      │    │  │
│  │   │                                                                  │    │  │
│  │   │  Format: Bincode                                                 │    │  │
│  │   └─────────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────┬───────────────────────────────────┘  │
│                                         │                                      │
│                                         ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                      DATASTREAMS TRANSFORMS                               │  │
│  │                                                                           │  │
│  │   Raw events are aggregated into derived state:                          │  │
│  │                                                                           │  │
│  │   hyperliquid.fills ───────┐                                             │  │
│  │   hyperliquid.funding ─────┼───► perps_position_aggregator              │  │
│  │   hyperliquid.liquidations─┘     │                                       │  │
│  │                                  ├──► hyperliquid.positions (RowBinary)  │  │
│  │                                  └──► hyperliquid.account_overview       │  │
│  │                                                                           │  │
│  │   Partition Key: user_id (ensures position consistency)                  │  │
│  └──────────────────────────────────────┬───────────────────────────────────┘  │
│                                         │                                      │
│                                         ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                          CLICKHOUSE CLUSTER                               │  │
│  │                                                                           │  │
│  │   ┌────────────────────────────────────────────────────────────────────┐ │  │
│  │   │                      NATS QUEUE TABLES                              │ │  │
│  │   │  (NATS Engine - Ephemeral ingestion buffers)                       │ │  │
│  │   │                                                                     │ │  │
│  │   │  perps_trades_queue              perps_funding_queue               │ │  │
│  │   │  perps_positions_queue           perps_account_overview_queue      │ │  │
│  │   │  hyperliquid_deposits_queue      hyperliquid_withdrawals_queue     │ │  │
│  │   │  hyperliquid_liquidations_queue                                    │ │  │
│  │   └────────────────────────────────────┬───────────────────────────────┘ │  │
│  │                                        │                                  │  │
│  │                         Materialized Views                                │  │
│  │                                        │                                  │  │
│  │   ┌────────────────────────────────────▼───────────────────────────────┐ │  │
│  │   │                      STORAGE TABLES                                 │ │  │
│  │   │                                                                     │ │  │
│  │   │  ┌─────────────────────┐  ┌─────────────────────────────────────┐  │ │  │
│  │   │  │   EVENT HISTORY     │  │         CURRENT STATE               │  │ │  │
│  │   │  │   (MergeTree)       │  │    (ReplacingMergeTree)             │  │ │  │
│  │   │  │                     │  │                                      │  │ │  │
│  │   │  │  • perps_trades     │  │  • perps_positions                  │  │ │  │
│  │   │  │  • perps_funding    │  │  • perps_account_overview           │  │ │  │
│  │   │  │  • deposits         │  │                                      │  │ │  │
│  │   │  │  • withdrawals      │  │         HISTORY                     │  │ │  │
│  │   │  │  • liquidations     │  │   (MergeTree with TTL)              │  │ │  │
│  │   │  │                     │  │                                      │  │ │  │
│  │   │  │                     │  │  • perps_positions_history          │  │ │  │
│  │   │  │                     │  │  • perps_account_history            │  │ │  │
│  │   │  └─────────────────────┘  └─────────────────────────────────────┘  │ │  │
│  │   └────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Differences from Solana Pipeline

| Aspect | Solana Spot | Hyperliquid Perps |
|--------|-------------|-------------------|
| **Serialization** | RowBinary | Bincode (raw events) → RowBinary (processed) |
| **Event Source** | On-chain transactions | WebSocket feeds + L1 events |
| **Settlement** | ~400ms blocks | Sub-second order book |
| **State Model** | Stateless trades | Stateful positions requiring aggregation |
| **Funding** | N/A | 8-hour payment intervals |
| **Transform Complexity** | Simple enrichment | State machine aggregation |

---

## 1. Trade Fills: The Atomic Unit of Perps Activity

Every perpetual futures trade generates a "fill" event when an order matches against the order book. Unlike spot swaps which are single atomic transactions, perps fills include critical additional metadata.

### Understanding Fill Events

A fill represents the execution of an order. Key concepts:

- **Side**: "A" (ask/sell) or "B" (bid/buy) - refers to which side of the order book
- **Direction**: Describes the position change - "Open Long", "Close Long", "Open Short", "Close Short"
- **Start Position**: The position size before this fill (crucial for P&L calculation)
- **Closed P&L**: Realized profit/loss when reducing a position (zero for position opens)

### Event Structure

```rust
/// Trade fill event from Hyperliquid
/// Source: apex-primitives/src/click_perps_events.rs
/// NATS Subject: hyperliquid.fills
/// Serialization: Bincode
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HyperliquidFill {
    /// Click platform user UUID (internal identifier)
    pub user_id: Uuid,

    /// User's EVM address on Hyperliquid (0x...)
    pub user_address: String,

    /// Trading pair coin symbol (e.g., "BTC", "ETH", "SOL", "HYPE")
    pub coin: String,

    /// Execution price as decimal string (e.g., "45000.50")
    /// Using String to preserve precision across Bincode serialization
    pub px: String,

    /// Trade size in contracts as decimal string (e.g., "0.1")
    pub sz: String,

    /// Order book side: "A" (ask/sell) or "B" (bid/buy)
    /// "A" = matched against ask side (you bought)
    /// "B" = matched against bid side (you sold)
    pub side: String,

    /// Timestamp in milliseconds since Unix epoch
    pub time: u64,

    /// Position size before this fill (for P&L calculation)
    pub start_position: String,

    /// Direction of position change:
    /// - "Open Long": Opening or adding to long position
    /// - "Close Long": Reducing long position
    /// - "Open Short": Opening or adding to short position
    /// - "Close Short": Reducing short position
    pub dir: String,

    /// Realized P&L from this fill (non-zero when closing positions)
    pub closed_pnl: String,

    /// Transaction hash for on-chain verification
    pub hash: String,

    /// Order ID (links multiple fills to same order)
    pub oid: u64,

    /// Trading fee paid for this fill
    pub fee: String,

    /// Unique trade ID (globally unique across all fills)
    pub tid: u64,
}
```

### Field-by-Field Analysis

| Field | Type | Description | Example | Analytics Use |
|-------|------|-------------|---------|---------------|
| `user_id` | UUID | Click internal user ID | `550e8400-e29b-...` | User identification |
| `user_address` | String | EVM address | `0x742d35Cc...` | Wallet analytics |
| `coin` | String | Trading pair | `"BTC"`, `"ETH"` | Market breakdown |
| `px` | String | Execution price | `"45000.50"` | P&L calculation |
| `sz` | String | Trade size | `"0.1"` | Volume tracking |
| `side` | String | Order book side | `"A"` (sell), `"B"` (buy) | Buy/sell ratio |
| `time` | UInt64 | Timestamp (ms) | `1704067200000` | Time series |
| `start_position` | String | Position before fill | `"0.5"` | Position changes |
| `dir` | String | Direction change | `"Open Long"` | Trade categorization |
| `closed_pnl` | String | Realized P&L | `"150.25"` | P&L attribution |
| `hash` | String | TX hash | `"0xabc123..."` | Verification |
| `oid` | UInt64 | Order ID | `123456789` | Order linking |
| `fee` | String | Trading fee | `"2.50"` | Cost tracking |
| `tid` | UInt64 | Trade ID | `987654321` | Deduplication |

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- PERPS TRADES TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Stores all Hyperliquid trade fills for history and analytics
-- Engine: MergeTree (immutable events, optimal for append-only workloads)
-- Partitioning: Monthly for efficient time-range queries
-- TTL: 1 year retention
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.perps_trades ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- IDENTIFIERS
    -- ═══════════════════════════════════════════════════════════════════════

    user_id UUID COMMENT 'Click platform user UUID',
    user_address String COMMENT 'EVM address (0x...)',
    coin LowCardinality(String) COMMENT 'Trading pair (BTC, ETH, HYPE)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TRADE EXECUTION DETAILS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Price and size as high-precision decimals
    -- Decimal(38, 18) supports 18 decimal places for wei-level precision
    px Decimal(38, 18) COMMENT 'Execution price',
    sz Decimal(38, 18) COMMENT 'Trade size in contracts',

    -- Side and direction for trade classification
    side LowCardinality(String) COMMENT 'Order book side: A (sell) or B (buy)',
    dir LowCardinality(String) COMMENT 'Direction: Open Long, Close Long, Open Short, Close Short',

    -- ═══════════════════════════════════════════════════════════════════════
    -- POSITION CONTEXT
    -- ═══════════════════════════════════════════════════════════════════════

    start_position Decimal(38, 18) COMMENT 'Position size before this fill',

    -- ═══════════════════════════════════════════════════════════════════════
    -- P&L AND FEES
    -- ═══════════════════════════════════════════════════════════════════════

    closed_pnl Decimal(38, 18) COMMENT 'Realized P&L from this fill (0 for opens)',
    fee Decimal(38, 18) COMMENT 'Trading fee paid',

    -- ═══════════════════════════════════════════════════════════════════════
    -- UNIQUE IDENTIFIERS FOR DEDUPLICATION
    -- ═══════════════════════════════════════════════════════════════════════

    hash String COMMENT 'Transaction hash for verification',
    oid UInt64 COMMENT 'Order ID (links fills to same order)',
    tid UInt64 COMMENT 'Trade ID (globally unique)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TIMESTAMPS
    -- ═══════════════════════════════════════════════════════════════════════

    time UInt64 COMMENT 'Execution timestamp in milliseconds',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS (ClickHouse Responsibility)
    -- ═══════════════════════════════════════════════════════════════════════

    -- Parse millisecond timestamp to DateTime64
    timestamp DateTime64(3) MATERIALIZED fromUnixTimestamp64Milli(time)
        COMMENT 'Parsed execution timestamp',

    -- Calculate trade volume in USD
    volume_usd Decimal(38, 18) MATERIALIZED px * sz
        COMMENT 'Trade notional value (price × size)',

    -- Human-readable side label
    side_label LowCardinality(String) MATERIALIZED if(side = 'B', 'BUY', 'SELL')
        COMMENT 'Human-readable side (BUY/SELL)',

    -- Trade date for partitioning
    trade_date Date MATERIALIZED toDate(timestamp)
        COMMENT 'Trade date for partition pruning',

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES FOR COMMON QUERY PATTERNS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Bloom filter for user lookups (reduces disk I/O significantly)
    INDEX idx_user user_id TYPE bloom_filter GRANULARITY 4,

    -- Bloom filter for coin filtering
    INDEX idx_coin coin TYPE bloom_filter GRANULARITY 4,

    -- Bloom filter for transaction hash lookups
    INDEX idx_hash hash TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/perps_trades', '{replica}')
PARTITION BY toYYYYMM(trade_date)
ORDER BY (user_id, coin, time, tid)
TTL trade_date + INTERVAL 365 DAY
COMMENT 'Hyperliquid trade fills. Immutable event log. TTL: 1 year.'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered_storage';
```

### Why MergeTree, Not ReplacingMergeTree?

Trade fills are **immutable events**. Once a fill occurs, it never changes. This fundamental characteristic drives our engine choice:

| Engine | Use Case | Trade Fills |
|--------|----------|-------------|
| MergeTree | Append-only events | Perfect fit |
| ReplacingMergeTree | Mutable state | Unnecessary overhead |
| AggregatingMergeTree | Pre-aggregation | Loses individual trades |

MergeTree provides:
- Optimal insert performance (no version tracking)
- Fastest reads (no FINAL needed)
- Best compression (predictable data patterns)

### Essential Trade Queries

**Recent trades by user:**
```sql
-- Get the last 50 trades for a user
-- Uses ORDER BY for efficient tail scan
SELECT
    timestamp,
    coin,
    side_label,
    dir AS direction,
    px AS price,
    sz AS size,
    volume_usd,
    closed_pnl,
    fee
FROM datastreams.perps_trades
WHERE user_id = {user_id:UUID}
ORDER BY time DESC
LIMIT 50;
```

**Realized P&L by coin (last 24 hours):**
```sql
-- Break down realized P&L by trading pair
-- Only closing trades contribute to realized P&L
SELECT
    coin,
    -- Trade counts by type
    countIf(dir LIKE 'Open%') AS opens,
    countIf(dir LIKE 'Close%') AS closes,

    -- P&L components
    sum(closed_pnl) AS realized_pnl,
    sum(fee) AS total_fees,
    sum(closed_pnl) - sum(fee) AS net_pnl,

    -- Volume metrics
    sum(volume_usd) AS total_volume,
    avg(volume_usd) AS avg_trade_size
FROM datastreams.perps_trades
WHERE user_id = {user_id:UUID}
  AND time >= toUnixTimestamp64Milli(now() - INTERVAL 24 HOUR)
GROUP BY coin
ORDER BY net_pnl DESC;
```

**Platform-wide volume leaderboard:**
```sql
-- Top 100 traders by volume today
-- Used for Click builder code attribution
SELECT
    user_address,
    count() AS trade_count,
    sum(volume_usd) AS total_volume,
    sum(closed_pnl) AS total_realized_pnl,
    sum(fee) AS total_fees,
    avg(volume_usd) AS avg_trade_size,

    -- Win rate calculation
    countIf(closed_pnl > 0) AS winning_trades,
    countIf(closed_pnl < 0) AS losing_trades,
    if(countIf(closed_pnl != 0) > 0,
       countIf(closed_pnl > 0) / countIf(closed_pnl != 0) * 100,
       0) AS win_rate_pct

FROM datastreams.perps_trades
WHERE trade_date = today()
GROUP BY user_address
HAVING trade_count >= 5  -- Minimum activity filter
ORDER BY total_volume DESC
LIMIT 100;
```

**Order execution analysis (fills per order):**
```sql
-- Analyze how orders are filled (single fill vs multiple)
-- Useful for understanding execution quality
SELECT
    oid AS order_id,
    coin,
    count() AS fill_count,
    sum(sz) AS total_size,
    min(px) AS min_price,
    max(px) AS max_price,
    -- VWAP calculation
    sum(px * sz) / sum(sz) AS vwap,
    -- Slippage from first fill
    abs(max(px) - min(px)) / min(px) * 100 AS price_slippage_pct,
    sum(fee) AS total_fees
FROM datastreams.perps_trades
WHERE user_id = {user_id:UUID}
  AND time >= toUnixTimestamp64Milli(now() - INTERVAL 7 DAY)
GROUP BY oid, coin
HAVING fill_count > 1  -- Only show multi-fill orders
ORDER BY fill_count DESC
LIMIT 50;
```

---

## 2. Position Tracking: Stateful P&L Analytics

Unlike trades which are point-in-time events, positions are **stateful objects** that evolve with each fill, funding payment, and market price movement. This requires a fundamentally different data modeling approach.

### Position State Components

A perpetual position tracks multiple dimensions simultaneously:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          POSITION STATE MODEL                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      CORE POSITION DATA                              │   │
│   │                                                                       │   │
│   │   Direction: LONG or SHORT                                           │   │
│   │   Size: 0.5 BTC                                                      │   │
│   │   Entry Price: $45,000 (VWAP of all fills)                          │   │
│   │   Leverage: 10x                                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│              ┌─────────────────────┴─────────────────────┐                  │
│              ▼                                           ▼                  │
│   ┌─────────────────────────┐             ┌─────────────────────────────┐  │
│   │   UNREALIZED P&L        │             │      REALIZED P&L           │  │
│   │                         │             │                              │  │
│   │   Mark Price: $46,000   │             │   Closed Trades: +$2,500    │  │
│   │   Unrealized: +$500     │             │   Funding Paid: -$150       │  │
│   │   ROE: +11.1%           │             │   Fees Paid: -$75           │  │
│   │                         │             │   Net Realized: +$2,275     │  │
│   │   (Fluctuates with      │             │                              │  │
│   │    market price)        │             │   (Locked in, won't change) │  │
│   └─────────────────────────┘             └─────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      RISK METRICS                                    │   │
│   │                                                                       │   │
│   │   Margin Used: $4,500 (position_value / leverage)                   │   │
│   │   Liquidation Price: $40,500 (price where margin = 0)               │   │
│   │   Distance to Liquidation: 10% ($4,500 buffer)                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Event Structure

```rust
/// A perpetual futures position for a Click user on Hyperliquid.
/// Source: Derived by DataStreams from fill and funding events
/// NATS Subject: hyperliquid.positions
/// Serialization: RowBinary (post-transformation)
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct PerpsPosition {
    /// Click user UUID
    pub user_id: Uuid,

    /// User's EVM address (0x...)
    pub user_address: String,

    /// Trading pair coin (e.g., "BTC", "ETH", "HYPE")
    pub coin: String,

    /// Position direction: "long" or "short"
    pub direction: String,

    /// Position size in contracts (absolute value)
    pub size: Decimal,

    /// Average entry price (VWAP of all opening fills)
    pub entry_price: Decimal,

    /// Current mark price from Hyperliquid
    /// Updated frequently for real-time P&L
    pub mark_price: Decimal,

    /// Unrealized P&L: (mark_price - entry_price) × size × direction_multiplier
    /// Fluctuates with market price
    pub unrealized_pnl: Decimal,

    /// Realized P&L: Sum of closed_pnl from all fills
    /// Locked in value that doesn't change
    pub realized_pnl: Decimal,

    /// Total P&L: unrealized + realized
    pub total_pnl: Decimal,

    /// Margin allocated to this position
    /// margin_used = position_value / leverage
    pub margin_used: Decimal,

    /// Position leverage (e.g., 10, 20, 50)
    pub leverage: Decimal,

    /// Liquidation price for this position
    /// Price at which remaining margin = 0
    pub liquidation_price: Decimal,

    /// Accumulated funding payments (positive = received, negative = paid)
    pub accumulated_funding: Decimal,

    /// Return on Equity: (unrealized_pnl / margin_used) × 100
    pub roe_pct: Decimal,

    /// Number of fills for this position
    pub trade_count: u64,

    /// Total fees paid for this position
    pub total_fees: Decimal,

    /// Position opened timestamp (first fill)
    pub opened_at: OffsetDateTime,

    /// Last update timestamp (latest fill/funding/price update)
    pub last_updated: OffsetDateTime,
}
```

### P&L Calculation Deep Dive

Understanding how P&L is calculated is crucial for correct implementation:

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- P&L CALCULATION FORMULAS
-- ═══════════════════════════════════════════════════════════════════════════

-- Direction multiplier: converts price difference to P&L
-- Long: profit when price goes UP
-- Short: profit when price goes DOWN
direction_multiplier = if(direction = 'long', 1, -1)

-- Unrealized P&L (mark-to-market)
-- This fluctuates continuously with the mark price
unrealized_pnl = (mark_price - entry_price) * size * direction_multiplier

-- Example:
-- Long 0.5 BTC at $45,000, mark price $46,000
-- = ($46,000 - $45,000) × 0.5 × 1 = +$500

-- Short 0.5 BTC at $45,000, mark price $46,000
-- = ($46,000 - $45,000) × 0.5 × (-1) = -$500

-- Return on Equity (ROE%)
-- Measures efficiency of margin usage
roe_pct = if(margin_used > 0, (unrealized_pnl / margin_used) * 100, 0)

-- Example:
-- Unrealized: +$500, Margin: $4,500
-- ROE = $500 / $4,500 × 100 = 11.1%

-- Liquidation Price (simplified formula)
-- Actual calculation depends on Hyperliquid's risk engine
-- For LONG: price at which loss = margin
liquidation_price_long = entry_price * (1 - (1 / leverage) * 0.9)  -- 0.9 = safety buffer

-- For SHORT: price at which loss = margin
liquidation_price_short = entry_price * (1 + (1 / leverage) * 0.9)

-- Example (10x Long at $45,000):
-- = $45,000 × (1 - 0.1 × 0.9) = $45,000 × 0.91 = $40,950

-- Net P&L (total actual profit/loss)
-- Includes all components
net_pnl = unrealized_pnl + realized_pnl + accumulated_funding - total_fees
```

### ClickHouse Implementation: The Dual-Table Pattern

For position data, we need two tables:
1. **Current State**: ReplacingMergeTree for latest position per user/coin
2. **History**: MergeTree for time-series snapshots (P&L charts)

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- TABLE 1: PERPS_POSITIONS (Current State)
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Stores the LATEST position state for each user-coin pair
-- Engine: ReplacingMergeTree - Automatically deduplicates to keep newest version
-- Key: (user_id, coin) - One row per position
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.perps_positions ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- PRIMARY IDENTIFIERS
    -- ═══════════════════════════════════════════════════════════════════════

    user_id UUID COMMENT 'Click user UUID',
    user_address String COMMENT 'EVM address (0x...)',
    coin LowCardinality(String) COMMENT 'Trading pair (BTC, ETH, HYPE)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- POSITION DETAILS
    -- ═══════════════════════════════════════════════════════════════════════

    direction LowCardinality(String) COMMENT 'long or short',
    size Decimal(38, 18) COMMENT 'Position size in contracts',
    entry_price Decimal(38, 18) COMMENT 'VWAP entry price',
    mark_price Decimal(38, 18) COMMENT 'Current mark price',

    -- Position value (notional)
    position_value Decimal(38, 18) MATERIALIZED mark_price * size
        COMMENT 'Notional value at current mark',

    -- ═══════════════════════════════════════════════════════════════════════
    -- P&L METRICS
    -- ═══════════════════════════════════════════════════════════════════════

    unrealized_pnl Decimal(38, 18) COMMENT 'Current unrealized P&L',
    realized_pnl Decimal(38, 18) COMMENT 'Cumulative realized P&L',
    total_pnl Decimal(38, 18) COMMENT 'unrealized + realized',

    -- ═══════════════════════════════════════════════════════════════════════
    -- RISK METRICS
    -- ═══════════════════════════════════════════════════════════════════════

    margin_used Decimal(38, 18) COMMENT 'Margin allocated to position',
    leverage Decimal(38, 18) COMMENT 'Position leverage (e.g., 10)',
    liquidation_price Decimal(38, 18) COMMENT 'Price triggering liquidation',

    -- Distance to liquidation as percentage
    liq_distance_pct Decimal(38, 18) MATERIALIZED
        if(direction = 'long',
           (mark_price - liquidation_price) / mark_price * 100,
           (liquidation_price - mark_price) / mark_price * 100)
        COMMENT 'Percentage distance to liquidation',

    -- ═══════════════════════════════════════════════════════════════════════
    -- FUNDING & FEES
    -- ═══════════════════════════════════════════════════════════════════════

    accumulated_funding Decimal(38, 18) COMMENT 'Net funding payments',
    funding_since_open Decimal(38, 18) COMMENT 'Funding since position opened',
    total_fees Decimal(38, 18) COMMENT 'Total trading fees paid',

    -- ═══════════════════════════════════════════════════════════════════════
    -- PERFORMANCE METRICS
    -- ═══════════════════════════════════════════════════════════════════════

    roe_pct Decimal(38, 18) COMMENT 'Return on equity %',
    trade_count UInt64 COMMENT 'Number of fills',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TIMESTAMPS
    -- ═══════════════════════════════════════════════════════════════════════

    opened_at DateTime64(9) COMMENT 'Position opened timestamp',
    last_updated DateTime64(9) COMMENT 'Last update timestamp',

    -- ═══════════════════════════════════════════════════════════════════════
    -- VERSION CONTROL FOR REPLACINGMERGETREE
    -- ═══════════════════════════════════════════════════════════════════════

    -- Nanosecond precision ensures unique versions
    version UInt64 MATERIALIZED toUnixTimestamp64Nano(last_updated)
        COMMENT 'Version for deduplication (newer wins)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED STATUS
    -- ═══════════════════════════════════════════════════════════════════════

    position_status LowCardinality(String) MATERIALIZED
        if(size > 0, 'OPEN', 'CLOSED')
        COMMENT 'OPEN or CLOSED based on size',

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES
    -- ═══════════════════════════════════════════════════════════════════════

    INDEX idx_user user_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_coin coin TYPE bloom_filter GRANULARITY 4,
    INDEX idx_status position_status TYPE set(10) GRANULARITY 1
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/perps_positions',
    '{replica}',
    version
)
ORDER BY (user_id, coin)
COMMENT 'Current perps position state. One row per user-coin pair.'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered_storage';
```

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- TABLE 2: PERPS_POSITIONS_HISTORY (Time-Series Snapshots)
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Stores ALL position snapshots for P&L charting
-- Engine: MergeTree - Keeps all records for historical analysis
-- Use Case: "Show me my BTC P&L over the last 7 days"
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.perps_positions_history ON CLUSTER `server-apex`
(
    -- Identifiers
    user_id UUID,
    user_address String,
    coin LowCardinality(String),

    -- Position state at snapshot time
    direction LowCardinality(String),
    size Decimal(38, 18),
    entry_price Decimal(38, 18),
    mark_price Decimal(38, 18),

    -- P&L at snapshot
    unrealized_pnl Decimal(38, 18),
    realized_pnl Decimal(38, 18),
    total_pnl Decimal(38, 18),
    roe_pct Decimal(38, 18),

    -- Risk at snapshot
    margin_used Decimal(38, 18),
    leverage Decimal(38, 18),
    liquidation_price Decimal(38, 18),

    -- Funding at snapshot
    accumulated_funding Decimal(38, 18),

    -- Snapshot timestamp
    snapshot_time DateTime64(9),
    snapshot_date Date MATERIALIZED toDate(snapshot_time),

    -- Indexes
    INDEX idx_user user_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_coin coin TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/perps_positions_history', '{replica}')
PARTITION BY toYYYYMM(snapshot_date)
ORDER BY (user_id, coin, snapshot_time)
TTL snapshot_date + INTERVAL 90 DAY
COMMENT 'Historical position snapshots for P&L graphs. TTL: 90 days.'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered_storage';
```

### Position Queries

**Current open positions with risk assessment:**
```sql
-- Get all open positions with risk metrics
SELECT
    coin,
    direction,
    size,
    entry_price,
    mark_price,

    -- P&L
    unrealized_pnl,
    realized_pnl,
    total_pnl,
    roe_pct,

    -- Risk
    leverage,
    liquidation_price,
    liq_distance_pct,

    -- Risk classification
    CASE
        WHEN liq_distance_pct < 5 THEN 'CRITICAL'
        WHEN liq_distance_pct < 10 THEN 'HIGH_RISK'
        WHEN liq_distance_pct < 25 THEN 'MODERATE'
        ELSE 'HEALTHY'
    END AS risk_level

FROM datastreams.perps_positions FINAL
WHERE user_id = {user_id:UUID}
  AND size > 0  -- Only open positions
ORDER BY abs(unrealized_pnl) DESC;
```

**Platform-wide positions at risk of liquidation:**
```sql
-- Find positions within 5% of liquidation
-- Useful for risk monitoring and potential cascade detection
SELECT
    user_address,
    coin,
    direction,
    size,
    mark_price,
    liquidation_price,
    liq_distance_pct,
    unrealized_pnl,
    leverage,
    margin_used
FROM datastreams.perps_positions FINAL
WHERE size > 0
  AND liq_distance_pct < 5  -- Within 5% of liquidation
ORDER BY liq_distance_pct ASC
LIMIT 100;
```

**P&L history for charting:**
```sql
-- Hourly P&L snapshots for chart rendering
-- Aggregates to reduce data points for frontend
SELECT
    toStartOfHour(snapshot_time) AS hour,
    coin,

    -- Use last value in each hour (most recent)
    argMax(unrealized_pnl, snapshot_time) AS unrealized_pnl,
    argMax(realized_pnl, snapshot_time) AS realized_pnl,
    argMax(total_pnl, snapshot_time) AS total_pnl,
    argMax(roe_pct, snapshot_time) AS roe_pct,
    argMax(mark_price, snapshot_time) AS mark_price,

    -- Also track min/max for range visualization
    min(unrealized_pnl) AS min_unrealized,
    max(unrealized_pnl) AS max_unrealized

FROM datastreams.perps_positions_history
WHERE user_id = {user_id:UUID}
  AND coin = {coin:String}
  AND snapshot_time >= now() - INTERVAL 7 DAY
GROUP BY hour, coin
ORDER BY hour;
```

---

## 3. Funding Payments: The Hidden Cost of Leverage

Perpetual futures maintain price parity with spot markets through **funding payments**. Every 8 hours, traders pay or receive funding based on the difference between the perpetual price and the spot index price.

### Understanding Funding Mechanics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          FUNDING RATE MECHANICS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Funding Rate = (Perp Price - Spot Index) / Spot Index × Time Factor     │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    POSITIVE FUNDING RATE                             │   │
│   │                    (Perp > Spot = Bullish)                          │   │
│   │                                                                       │   │
│   │   ┌─────────────┐                     ┌─────────────┐               │   │
│   │   │    LONGS    │ ────── PAY ──────▶  │   SHORTS   │               │   │
│   │   │  (majority) │                     │ (minority)  │               │   │
│   │   └─────────────┘                     └─────────────┘               │   │
│   │                                                                       │   │
│   │   Effect: Encourages shorts, discourages longs                       │   │
│   │   Result: Perp price converges to spot                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    NEGATIVE FUNDING RATE                             │   │
│   │                    (Perp < Spot = Bearish)                          │   │
│   │                                                                       │   │
│   │   ┌─────────────┐                     ┌─────────────┐               │   │
│   │   │   SHORTS    │ ────── PAY ──────▶  │   LONGS    │               │   │
│   │   │  (majority) │                     │ (minority)  │               │   │
│   │   └─────────────┘                     └─────────────┘               │   │
│   │                                                                       │   │
│   │   Effect: Encourages longs, discourages shorts                       │   │
│   │   Result: Perp price converges to spot                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   Funding occurs every 8 hours: 00:00, 08:00, 16:00 UTC                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Event Structure

```rust
/// Funding payment event from Hyperliquid
/// Source: apex-primitives/src/click_perps_events.rs
/// NATS Subject: hyperliquid.funding
/// Serialization: Bincode
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HyperliquidFunding {
    /// Click user UUID
    pub user_id: Uuid,

    /// User's EVM address
    pub user_address: String,

    /// Trading pair (e.g., "BTC", "ETH")
    pub coin: String,

    /// Funding rate for this period (annualized)
    /// Typically between -0.1% and +0.1% per 8h period
    pub funding_rate: String,

    /// Funding payment amount
    /// Positive = received (you were paid)
    /// Negative = paid (you paid funding)
    pub payment: String,

    /// Timestamp of funding payment (ms)
    pub time: u64,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- PERPS FUNDING TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Stores all funding payment history
-- Engine: MergeTree (immutable events every 8 hours)
-- Use Case: Track cumulative funding costs/income per position
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.perps_funding ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- IDENTIFIERS
    -- ═══════════════════════════════════════════════════════════════════════

    user_id UUID COMMENT 'Click user UUID',
    user_address String COMMENT 'EVM address (0x...)',
    coin LowCardinality(String) COMMENT 'Trading pair (BTC, ETH, HYPE)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- FUNDING DETAILS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Funding rate as decimal (e.g., 0.0001 = 0.01%)
    funding_rate Decimal(38, 18) COMMENT 'Funding rate for this period',

    -- Payment amount (positive = received, negative = paid)
    payment Decimal(38, 18) COMMENT 'Payment: positive=received, negative=paid',

    -- Timestamp
    time UInt64 COMMENT 'Timestamp in milliseconds',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    timestamp DateTime64(3) MATERIALIZED fromUnixTimestamp64Milli(time)
        COMMENT 'Parsed timestamp',

    funding_date Date MATERIALIZED toDate(timestamp)
        COMMENT 'Date for partitioning',

    -- Human-readable payment direction
    payment_direction LowCardinality(String) MATERIALIZED
        if(payment >= 0, 'RECEIVED', 'PAID')
        COMMENT 'RECEIVED or PAID',

    -- Absolute payment amount (for aggregation)
    payment_abs Decimal(38, 18) MATERIALIZED abs(payment)
        COMMENT 'Absolute payment amount',

    -- Annualized funding rate (rate × 3 × 365)
    annual_rate_pct Decimal(38, 18) MATERIALIZED funding_rate * 3 * 365 * 100
        COMMENT 'Annualized funding rate as percentage',

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES
    -- ═══════════════════════════════════════════════════════════════════════

    INDEX idx_user user_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_coin coin TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/perps_funding', '{replica}')
PARTITION BY toYYYYMM(funding_date)
ORDER BY (user_id, coin, time)
TTL funding_date + INTERVAL 365 DAY
COMMENT 'Hyperliquid funding payments. Occurs every 8 hours. TTL: 1 year.'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered_storage';
```

### Funding Analytics

**Cumulative funding by coin:**
```sql
-- Track funding costs/income per position
SELECT
    coin,
    count() AS funding_periods,

    -- Total received (positive payments)
    sumIf(payment, payment > 0) AS total_received,

    -- Total paid (negative payments, shown as positive)
    sumIf(abs(payment), payment < 0) AS total_paid,

    -- Net funding (received - paid)
    sum(payment) AS net_funding,

    -- Average funding rate
    avg(funding_rate) AS avg_funding_rate,
    avg(annual_rate_pct) AS avg_annual_rate_pct,

    -- Funding volatility
    stddevPop(funding_rate) AS funding_rate_stddev

FROM datastreams.perps_funding
WHERE user_id = {user_id:UUID}
GROUP BY coin
ORDER BY net_funding DESC;
```

**Funding rate history for market analysis:**
```sql
-- Daily funding rate summary by coin
-- Useful for identifying market sentiment
SELECT
    toDate(timestamp) AS date,
    coin,

    -- Rate statistics
    avg(funding_rate) AS avg_rate,
    min(funding_rate) AS min_rate,
    max(funding_rate) AS max_rate,

    -- Payment statistics
    sum(payment) AS net_funding,
    count() AS payment_count,

    -- Sentiment indicator
    CASE
        WHEN avg(funding_rate) > 0.0001 THEN 'BULLISH'  -- Longs paying
        WHEN avg(funding_rate) < -0.0001 THEN 'BEARISH' -- Shorts paying
        ELSE 'NEUTRAL'
    END AS market_sentiment

FROM datastreams.perps_funding
WHERE time >= toUnixTimestamp64Milli(now() - INTERVAL 30 DAY)
GROUP BY date, coin
ORDER BY date DESC, coin;
```

**Top funding earners (contrarian traders):**
```sql
-- Find traders who consistently receive funding
-- These are often contrarian traders taking the minority side
SELECT
    user_address,
    count(DISTINCT coin) AS coins_traded,
    count() AS funding_periods,

    -- Funding P&L
    sum(payment) AS net_funding,
    sumIf(payment, payment > 0) AS total_received,
    sumIf(abs(payment), payment < 0) AS total_paid,

    -- Average per period
    avg(payment) AS avg_payment_per_period

FROM datastreams.perps_funding
WHERE funding_date >= today() - 30
  AND payment > 0  -- Only positive payments
GROUP BY user_address
ORDER BY net_funding DESC
LIMIT 50;
```

---

## 4. Account Overview: Portfolio-Level Analytics

While positions track individual coin exposure, account overview provides a **holistic view** of a trader's entire portfolio. This is the data that powers the "Portfolio" page in the UI.

### Account Overview Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ACCOUNT OVERVIEW STRUCTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      EQUITY & MARGIN                                 │   │
│   │                                                                       │   │
│   │   Margin Balance: $10,000  (deposits - withdrawals + realized)      │   │
│   │   Unrealized P&L: +$2,500  (sum across all positions)               │   │
│   │   Account Equity: $12,500  (margin + unrealized)                    │   │
│   │   Available Margin: $5,000 (equity - used margin)                   │   │
│   │   Used Margin: $7,500      (margin locked in positions)             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      RISK METRICS                                    │   │
│   │                                                                       │   │
│   │   Cross Margin Ratio: 60%  (used_margin / account_equity)           │   │
│   │                                                                       │   │
│   │   Risk Classification:                                               │   │
│   │   ┌────────────────────────────────────────────────────────────┐    │   │
│   │   │ HEALTHY    │ MODERATE  │ HIGH_RISK │ CRITICAL              │    │   │
│   │   │ < 50%      │ 50-80%    │ 80-90%    │ > 90%                 │    │   │
│   │   └────────────────────────────────────────────────────────────┘    │   │
│   │                                  ▲                                   │   │
│   │                                  │                                   │   │
│   │                            Current: 60%                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      CUMULATIVE METRICS                              │   │
│   │                                                                       │   │
│   │   Total Realized P&L: +$15,000                                      │   │
│   │   Total Funding Received: +$500                                      │   │
│   │   Total Funding Paid: -$200                                          │   │
│   │   Net Funding: +$300                                                 │   │
│   │   Total Fees Paid: -$750                                             │   │
│   │   Total Deposits: $50,000                                            │   │
│   │   Total Withdrawals: -$40,000                                        │   │
│   │   Net Deposits: $10,000                                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      ACTIVITY METRICS                                │   │
│   │                                                                       │   │
│   │   Open Positions: 3                                                  │   │
│   │   Total Trades: 1,247                                                │   │
│   │   Total Volume: $2.5M                                                │   │
│   │   First Activity: 2024-01-15                                         │   │
│   │   Last Activity: 2024-06-20                                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Event Structure

```rust
/// Account-level overview for a Click user on Hyperliquid
/// Source: Aggregated by DataStreams from all event types
/// NATS Subject: hyperliquid.account_overview
/// Serialization: RowBinary
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct PerpsAccountOverview {
    // ═══════════════════════════════════════════════════════════════════════
    // IDENTIFIERS
    // ═══════════════════════════════════════════════════════════════════════

    /// Click user UUID
    pub user_id: Uuid,

    /// User's EVM address
    pub user_address: String,

    // ═══════════════════════════════════════════════════════════════════════
    // BALANCE & EQUITY
    // ═══════════════════════════════════════════════════════════════════════

    /// Margin balance: deposits - withdrawals + realized_pnl + funding - fees
    pub margin_balance: Decimal,

    /// Total unrealized P&L across all open positions
    pub unrealized_pnl: Decimal,

    /// Account equity: margin_balance + unrealized_pnl
    pub account_equity: Decimal,

    /// Available margin for new positions
    pub available_margin: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // MARGIN USAGE & RISK
    // ═══════════════════════════════════════════════════════════════════════

    /// Total margin used across all positions
    pub used_margin: Decimal,

    /// Cross margin ratio: used_margin / account_equity
    /// Higher = closer to liquidation
    pub cross_margin_ratio: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // CUMULATIVE P&L
    // ═══════════════════════════════════════════════════════════════════════

    /// Total realized P&L (all-time)
    pub total_realized_pnl: Decimal,

    /// Total P&L: realized + unrealized
    pub total_pnl: Decimal,

    /// ROE%: total_pnl / total_deposits × 100
    pub roe_pct: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // FUNDING
    // ═══════════════════════════════════════════════════════════════════════

    /// Total funding received (positive)
    pub total_funding_received: Decimal,

    /// Total funding paid (as positive number)
    pub total_funding_paid: Decimal,

    /// Net funding: received - paid
    pub net_funding: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // DEPOSITS & WITHDRAWALS
    // ═══════════════════════════════════════════════════════════════════════

    /// Total USDC deposited (all-time)
    pub total_deposits: Decimal,

    /// Total USDC withdrawn (all-time)
    pub total_withdrawals: Decimal,

    /// Net deposits: deposits - withdrawals
    pub net_deposits: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // ACTIVITY METRICS
    // ═══════════════════════════════════════════════════════════════════════

    /// Total trading fees paid
    pub total_fees_paid: Decimal,

    /// Total number of trades
    pub total_trade_count: u64,

    /// Number of currently open positions
    pub open_positions_count: u64,

    /// Total trading volume (all-time)
    pub total_volume: Decimal,

    // ═══════════════════════════════════════════════════════════════════════
    // TIMESTAMPS
    // ═══════════════════════════════════════════════════════════════════════

    /// First activity timestamp
    pub first_activity: OffsetDateTime,

    /// Last update timestamp
    pub last_updated: OffsetDateTime,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- PERPS ACCOUNT OVERVIEW TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Portfolio-level aggregation per user
-- Engine: ReplacingMergeTree - One row per user, always latest state
-- Updates: On every fill, funding, deposit, withdrawal
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.perps_account_overview ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- IDENTIFIERS
    -- ═══════════════════════════════════════════════════════════════════════

    user_id UUID COMMENT 'Click user UUID',
    user_address String COMMENT 'EVM address (0x...)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- BALANCE & EQUITY
    -- ═══════════════════════════════════════════════════════════════════════

    margin_balance Decimal(38, 18) COMMENT 'Current margin balance',
    unrealized_pnl Decimal(38, 18) COMMENT 'Total unrealized P&L',
    account_equity Decimal(38, 18) COMMENT 'margin + unrealized',
    available_margin Decimal(38, 18) COMMENT 'Available for new positions',

    -- ═══════════════════════════════════════════════════════════════════════
    -- MARGIN USAGE & RISK
    -- ═══════════════════════════════════════════════════════════════════════

    used_margin Decimal(38, 18) COMMENT 'Total margin in use',
    cross_margin_ratio Decimal(38, 18) COMMENT 'used / equity',

    -- ═══════════════════════════════════════════════════════════════════════
    -- CUMULATIVE P&L
    -- ═══════════════════════════════════════════════════════════════════════

    total_realized_pnl Decimal(38, 18) COMMENT 'All-time realized P&L',
    total_pnl Decimal(38, 18) COMMENT 'realized + unrealized',
    roe_pct Decimal(38, 18) COMMENT 'Return on equity %',

    -- ═══════════════════════════════════════════════════════════════════════
    -- FUNDING
    -- ═══════════════════════════════════════════════════════════════════════

    total_funding_received Decimal(38, 18) COMMENT 'Total funding received',
    total_funding_paid Decimal(38, 18) COMMENT 'Total funding paid',
    net_funding Decimal(38, 18) COMMENT 'received - paid',

    -- ═══════════════════════════════════════════════════════════════════════
    -- DEPOSITS & WITHDRAWALS
    -- ═══════════════════════════════════════════════════════════════════════

    total_deposits Decimal(38, 18) COMMENT 'Total USDC deposited',
    total_withdrawals Decimal(38, 18) COMMENT 'Total USDC withdrawn',
    net_deposits Decimal(38, 18) COMMENT 'deposits - withdrawals',

    -- ═══════════════════════════════════════════════════════════════════════
    -- ACTIVITY METRICS
    -- ═══════════════════════════════════════════════════════════════════════

    total_fees_paid Decimal(38, 18) COMMENT 'Total trading fees',
    total_trade_count UInt64 COMMENT 'Total number of trades',
    open_positions_count UInt64 COMMENT 'Currently open positions',
    total_volume Decimal(38, 18) COMMENT 'All-time trading volume',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TIMESTAMPS
    -- ═══════════════════════════════════════════════════════════════════════

    first_activity DateTime64(9) COMMENT 'First activity timestamp',
    last_updated DateTime64(9) COMMENT 'Last update timestamp',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    version UInt64 MATERIALIZED toUnixTimestamp64Nano(last_updated),

    -- Risk level classification
    risk_level LowCardinality(String) MATERIALIZED
        CASE
            WHEN cross_margin_ratio >= 0.9 THEN 'CRITICAL'
            WHEN cross_margin_ratio >= 0.8 THEN 'HIGH_RISK'
            WHEN cross_margin_ratio >= 0.5 THEN 'MODERATE'
            ELSE 'HEALTHY'
        END
        COMMENT 'Risk classification based on margin ratio',

    -- Net trading performance (everything combined)
    net_performance Decimal(38, 18) MATERIALIZED
        total_realized_pnl + unrealized_pnl + net_funding - total_fees_paid
        COMMENT 'Net P&L after funding and fees',

    -- Account age in days
    account_age_days UInt32 MATERIALIZED dateDiff('day', first_activity, last_updated)
        COMMENT 'Days since first activity',

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES
    -- ═══════════════════════════════════════════════════════════════════════

    INDEX idx_address user_address TYPE bloom_filter GRANULARITY 4,
    INDEX idx_risk risk_level TYPE set(10) GRANULARITY 1
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/perps_account_overview',
    '{replica}',
    version
)
ORDER BY user_id
COMMENT 'Perps account overview. One row per user with latest state.'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered_storage';
```

### Account Analytics

**Portfolio dashboard query:**
```sql
-- Complete portfolio summary for UI
SELECT
    -- Equity breakdown
    account_equity,
    margin_balance,
    unrealized_pnl,
    available_margin,
    used_margin,

    -- Risk assessment
    cross_margin_ratio,
    risk_level,

    -- Performance metrics
    total_realized_pnl,
    total_pnl,
    roe_pct,
    net_performance,

    -- Funding breakdown
    total_funding_received,
    total_funding_paid,
    net_funding,

    -- Costs
    total_fees_paid,

    -- Activity
    open_positions_count,
    total_trade_count,
    total_volume,

    -- Account info
    account_age_days,
    first_activity,
    last_updated

FROM datastreams.perps_account_overview FINAL
WHERE user_id = {user_id:UUID};
```

**Platform-wide accounts by risk level:**
```sql
-- Monitor account health across platform
SELECT
    risk_level,
    count() AS account_count,

    -- Exposure metrics
    sum(used_margin) AS total_margin_at_risk,
    sum(account_equity) AS total_equity,
    avg(cross_margin_ratio) AS avg_margin_ratio,

    -- P&L
    sum(unrealized_pnl) AS total_unrealized_pnl,
    sum(total_realized_pnl) AS total_realized_pnl,

    -- Positions
    sum(open_positions_count) AS total_open_positions

FROM datastreams.perps_account_overview FINAL
WHERE open_positions_count > 0  -- Only accounts with exposure
GROUP BY risk_level
ORDER BY
    CASE risk_level
        WHEN 'CRITICAL' THEN 1
        WHEN 'HIGH_RISK' THEN 2
        WHEN 'MODERATE' THEN 3
        ELSE 4
    END;
```

**Trader leaderboard by net performance:**
```sql
-- Top traders by net P&L
SELECT
    user_address,
    account_equity,
    total_realized_pnl,
    unrealized_pnl,
    net_performance,
    roe_pct,
    total_volume,
    total_trade_count,
    account_age_days,

    -- Volume per trade (execution quality)
    total_volume / nullif(total_trade_count, 0) AS avg_trade_size

FROM datastreams.perps_account_overview FINAL
WHERE account_equity >= 1000  -- Minimum equity filter
  AND total_trade_count >= 10  -- Minimum activity
ORDER BY net_performance DESC
LIMIT 100;
```

---

## 5. Raw L1 Events: Deposits, Withdrawals, Liquidations

Beyond trading activity, we capture L1 events from Arbitrum for complete audit trails and risk analysis.

### 5.1 Deposits

Tracks funds moving from L1 (Arbitrum) to Hyperliquid.

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- HYPERLIQUID DEPOSITS TABLE
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.hyperliquid_deposits ON CLUSTER `server-apex`
(
    -- User identifier
    user String COMMENT 'User EVM address',

    -- Deposit details
    coin LowCardinality(String) COMMENT 'Deposited asset (usually USDC)',
    amount Decimal(38, 18) COMMENT 'Deposit amount',

    -- L1 transaction info
    tx_hash String COMMENT 'L1 transaction hash',
    l1_block_number UInt64 COMMENT 'Arbitrum block number',
    l1_timestamp DateTime64(3) COMMENT 'L1 block timestamp',

    -- Status tracking
    status Enum8('PENDING' = 1, 'CONFIRMED' = 2, 'FAILED' = 3)
        COMMENT 'Deposit status',

    -- Server timestamp
    server_timestamp DateTime64(9),
    _inserted_at_ch DateTime64(3) DEFAULT now64(3),

    -- Indexes
    INDEX idx_user user TYPE bloom_filter GRANULARITY 4,
    INDEX idx_status status TYPE set(10) GRANULARITY 1
)
ENGINE = ReplicatedReplacingMergeTree(_inserted_at_ch)
ORDER BY (user, tx_hash)
COMMENT 'L1 deposits to Hyperliquid.'
SETTINGS storage_policy = 'tiered_storage';
```

### 5.2 Withdrawals

Tracks funds moving from Hyperliquid back to L1.

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- HYPERLIQUID WITHDRAWALS TABLE
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.hyperliquid_withdrawals ON CLUSTER `server-apex`
(
    -- User identifier
    user String COMMENT 'User EVM address',

    -- Withdrawal details
    coin LowCardinality(String) COMMENT 'Withdrawn asset',
    amount Decimal(38, 18) COMMENT 'Withdrawal amount',
    destination String COMMENT 'Destination address',
    fee Decimal(38, 18) COMMENT 'Withdrawal fee',
    nonce UInt64 COMMENT 'Withdrawal nonce (unique per user)',

    -- Status tracking
    status Enum8('PENDING' = 1, 'QUEUED' = 2, 'CONFIRMED' = 3, 'FAILED' = 4)
        COMMENT 'Withdrawal status',

    -- Transaction info (null until confirmed)
    tx_hash Nullable(String) COMMENT 'L1 transaction hash',

    -- Timestamps
    requested_at DateTime64(3) COMMENT 'When withdrawal was requested',
    processed_at Nullable(DateTime64(3)) COMMENT 'When withdrawal was processed',
    server_timestamp DateTime64(9),
    _inserted_at_ch DateTime64(3) DEFAULT now64(3),

    -- Computed: processing time
    processing_time_seconds Int32 MATERIALIZED
        if(processed_at IS NOT NULL,
           dateDiff('second', requested_at, processed_at),
           NULL)
        COMMENT 'Time to process withdrawal',

    -- Indexes
    INDEX idx_user user TYPE bloom_filter GRANULARITY 4,
    INDEX idx_status status TYPE set(10) GRANULARITY 1
)
ENGINE = ReplicatedReplacingMergeTree(_inserted_at_ch)
ORDER BY (user, nonce)
COMMENT 'Withdrawals from Hyperliquid to L1.'
SETTINGS storage_policy = 'tiered_storage';
```

### 5.3 Liquidations

Tracks forced position closures when margin is insufficient.

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- HYPERLIQUID LIQUIDATIONS TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Critical for: Risk monitoring, cascade detection, market health analysis

CREATE TABLE datastreams.hyperliquid_liquidations ON CLUSTER `server-apex`
(
    -- Who got liquidated
    user String COMMENT 'Liquidated user address',
    coin LowCardinality(String) COMMENT 'Position coin',

    -- Position details at liquidation
    side Enum8('LONG' = 1, 'SHORT' = 2) COMMENT 'Position direction',
    size Decimal(38, 18) COMMENT 'Position size liquidated',
    price Decimal(38, 18) COMMENT 'Liquidation execution price',
    notional_value Decimal(38, 18) COMMENT 'Notional value liquidated',

    -- Who liquidated (may be null for auto-liquidation)
    liquidator Nullable(String) COMMENT 'Liquidator address (if any)',

    -- Financial impact
    margin_used Decimal(38, 18) COMMENT 'Margin that was in position',
    pnl Decimal(38, 18) COMMENT 'P&L at liquidation (usually negative)',
    fee Decimal(38, 18) COMMENT 'Liquidation fee',

    -- Market conditions at liquidation
    mark_price Decimal(38, 18) COMMENT 'Mark price at liquidation',
    oracle_price Decimal(38, 18) COMMENT 'Oracle price at liquidation',
    leverage Decimal(38, 18) COMMENT 'Leverage at liquidation',

    -- Account state
    account_value_before Decimal(38, 18) COMMENT 'Account value before liquidation',
    account_value_after Decimal(38, 18) COMMENT 'Account value after liquidation',

    -- Timestamps
    timestamp DateTime64(3) COMMENT 'Liquidation timestamp',
    server_timestamp DateTime64(9),
    _inserted_at_ch DateTime64(3) DEFAULT now64(3),

    -- Computed fields
    loss_amount Decimal(38, 18) MATERIALIZED abs(pnl)
        COMMENT 'Loss amount (always positive)',

    liquidation_date Date MATERIALIZED toDate(timestamp),

    -- Indexes
    INDEX idx_user user TYPE bloom_filter GRANULARITY 4,
    INDEX idx_coin coin TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree()
PARTITION BY toYYYYMM(liquidation_date)
ORDER BY (coin, timestamp, user)
TTL toDateTime(timestamp) + INTERVAL 365 DAY
COMMENT 'Position liquidations. Critical for risk analysis. TTL: 1 year.'
SETTINGS storage_policy = 'tiered_storage';
```

### Liquidation Analytics

**Liquidation cascade detection:**
```sql
-- Detect liquidation cascades (5+ liquidations per minute)
-- These indicate market stress and potential price impact
SELECT
    toStartOfMinute(timestamp) AS minute,
    coin,
    count() AS liquidations,
    sum(notional_value) AS cascade_volume,
    avg(leverage) AS avg_leverage,

    -- Price movement during cascade
    min(price) AS min_price,
    max(price) AS max_price,
    (max(price) - min(price)) / min(price) * 100 AS price_swing_pct

FROM datastreams.hyperliquid_liquidations
WHERE liquidation_date = today()
GROUP BY minute, coin
HAVING liquidations >= 5  -- Cascade threshold
ORDER BY cascade_volume DESC;
```

**Hourly liquidation heatmap:**
```sql
-- Identify hours with highest liquidation activity
-- Useful for understanding market volatility patterns
SELECT
    toHour(timestamp) AS hour_utc,
    coin,
    count() AS liquidations,
    sum(notional_value) AS total_volume,
    avg(leverage) AS avg_leverage,
    countIf(side = 'LONG') AS long_liquidations,
    countIf(side = 'SHORT') AS short_liquidations
FROM datastreams.hyperliquid_liquidations
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY hour_utc, coin
ORDER BY hour_utc, total_volume DESC;
```

---

## 6. Combined Analytics: Cross-Table Queries

### User Performance Dashboard

```sql
-- Complete user performance analysis
-- Combines trades, positions, funding, and account data
WITH
    -- Trade activity (last 30 days)
    trades AS (
        SELECT
            count() AS trade_count,
            sum(volume_usd) AS trading_volume,
            sum(closed_pnl) AS realized_from_trades,
            sum(fee) AS trading_fees,
            countIf(closed_pnl > 0) AS winning_trades,
            countIf(closed_pnl < 0) AS losing_trades
        FROM datastreams.perps_trades
        WHERE user_id = {user_id:UUID}
          AND trade_date >= today() - 30
    ),

    -- Funding (last 30 days)
    funding AS (
        SELECT
            count() AS funding_periods,
            sum(payment) AS net_funding,
            sumIf(payment, payment > 0) AS funding_received,
            sumIf(abs(payment), payment < 0) AS funding_paid
        FROM datastreams.perps_funding
        WHERE user_id = {user_id:UUID}
          AND funding_date >= today() - 30
    ),

    -- Current account state
    account AS (
        SELECT
            account_equity,
            unrealized_pnl,
            cross_margin_ratio,
            risk_level,
            open_positions_count,
            total_realized_pnl AS all_time_realized
        FROM datastreams.perps_account_overview FINAL
        WHERE user_id = {user_id:UUID}
    ),

    -- Current positions
    positions AS (
        SELECT
            count() AS position_count,
            sum(position_value) AS total_exposure,
            sum(unrealized_pnl) AS positions_unrealized
        FROM datastreams.perps_positions FINAL
        WHERE user_id = {user_id:UUID}
          AND size > 0
    )

SELECT
    -- Trading activity
    trades.trade_count,
    trades.trading_volume,
    trades.realized_from_trades,
    trades.trading_fees,
    trades.winning_trades,
    trades.losing_trades,
    if(trades.winning_trades + trades.losing_trades > 0,
       trades.winning_trades / (trades.winning_trades + trades.losing_trades) * 100,
       0) AS win_rate_pct,

    -- Funding
    funding.net_funding AS funding_30d,

    -- 30-day net P&L
    trades.realized_from_trades + funding.net_funding - trades.trading_fees AS net_pnl_30d,

    -- Current state
    account.account_equity,
    account.unrealized_pnl,
    account.cross_margin_ratio,
    account.risk_level,
    account.open_positions_count,

    -- Positions
    positions.total_exposure,
    positions.positions_unrealized

FROM trades, funding, account, positions;
```

### Market Health Overview

```sql
-- Platform-wide market health metrics
SELECT
    coin,

    -- Position metrics (from current positions)
    countIf(direction = 'long') AS long_positions,
    countIf(direction = 'short') AS short_positions,
    sumIf(position_value, direction = 'long') AS long_exposure,
    sumIf(position_value, direction = 'short') AS short_exposure,

    -- Long/short ratio
    long_exposure / nullif(short_exposure, 0) AS long_short_ratio,

    -- Liquidation risk
    countIf(liq_distance_pct < 10) AS at_risk_positions,
    sumIf(position_value, liq_distance_pct < 10) AS at_risk_value,

    -- Trading activity (24h)
    trades.trade_count_24h,
    trades.volume_24h,

    -- Liquidations (24h)
    liqs.liquidation_count_24h,
    liqs.liquidated_value_24h

FROM datastreams.perps_positions FINAL AS p

LEFT JOIN (
    SELECT
        coin,
        count() AS trade_count_24h,
        sum(volume_usd) AS volume_24h
    FROM datastreams.perps_trades
    WHERE trade_date = today()
       OR (trade_date = today() - 1 AND timestamp >= now() - INTERVAL 24 HOUR)
    GROUP BY coin
) AS trades ON p.coin = trades.coin

LEFT JOIN (
    SELECT
        coin,
        count() AS liquidation_count_24h,
        sum(notional_value) AS liquidated_value_24h
    FROM datastreams.hyperliquid_liquidations
    WHERE timestamp >= now() - INTERVAL 24 HOUR
    GROUP BY coin
) AS liqs ON p.coin = liqs.coin

WHERE p.size > 0
GROUP BY p.coin, trades.trade_count_24h, trades.volume_24h,
         liqs.liquidation_count_24h, liqs.liquidated_value_24h
ORDER BY long_exposure + short_exposure DESC;
```

---

## 7. Performance Optimization

### Indexes for Common Query Patterns

```sql
-- User lookup optimization (most common query pattern)
ALTER TABLE datastreams.perps_trades
ADD INDEX idx_user_time (user_id, time) TYPE minmax GRANULARITY 4;

-- Coin filtering for market analysis
ALTER TABLE datastreams.perps_positions
ADD INDEX idx_coin_size (coin, size) TYPE minmax GRANULARITY 4;

-- Risk level filtering for monitoring
ALTER TABLE datastreams.perps_account_overview
ADD INDEX idx_risk_margin (risk_level, cross_margin_ratio) TYPE minmax GRANULARITY 4;
```

### Materialized Views for Aggregation

```sql
-- Hourly volume aggregation (reduces query load for dashboards)
CREATE MATERIALIZED VIEW datastreams.mv_perps_volume_hourly
ENGINE = SummingMergeTree()
ORDER BY (coin, hour)
AS SELECT
    coin,
    toStartOfHour(timestamp) AS hour,
    sum(volume_usd) AS volume,
    count() AS trades,
    sum(fee) AS fees,
    uniq(user_id) AS unique_traders,
    sum(closed_pnl) AS realized_pnl
FROM datastreams.perps_trades
GROUP BY coin, hour;

-- Query the materialized view instead of raw data
SELECT
    coin,
    hour,
    volume,
    trades,
    unique_traders
FROM datastreams.mv_perps_volume_hourly
WHERE hour >= now() - INTERVAL 24 HOUR
ORDER BY hour, volume DESC;
```

---

## Summary

Hyperliquid perpetual futures analytics extend our platform with a complete derivatives trading data model:

| Component | Table Engine | Purpose |
|-----------|--------------|---------|
| Trade Fills | MergeTree | Immutable trade history |
| Positions | ReplacingMergeTree | Current position state |
| Position History | MergeTree | P&L time series |
| Funding | MergeTree | 8-hour payment history |
| Account Overview | ReplacingMergeTree | Portfolio aggregation |
| Deposits | ReplacingMergeTree | L1 deposit tracking |
| Withdrawals | ReplacingMergeTree | Withdrawal tracking |
| Liquidations | MergeTree | Forced closure events |

The patterns from spot DEX analytics translate directly:
- **MergeTree** for immutable events (trades, funding, liquidations)
- **ReplacingMergeTree** for mutable state (positions, accounts)
- **FINAL** keyword for reading current state
- **Materialized columns** for derived metrics
- **Bloom filter indexes** for high-cardinality lookups

---

## Next in the Series

**Part 8: Risk Analysis, Wallet Funding Sources & External Data** - We'll explore wallet funding source tracking for on-chain risk analysis, token meta classification, and integrating external data sources (GoPlus, RugCheck) for comprehensive risk scoring.
