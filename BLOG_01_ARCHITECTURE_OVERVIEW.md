# Building Real-Time Solana DEX Analytics with ClickHouse

## Part 1: Architecture Overview & Stream Processing

**Series Overview:** This 6-part technical blog series documents how we built a real-time analytics system processing 100,000+ Solana DEX trades per second, generating OHLCV candles across 30 timeframes, calculating FIFO-based PnL for millions of positions, and serving sub-200ms queries to power our trading platform.

---

## Table of Contents

1. [The Challenge](#the-challenge)
2. [Architecture Overview](#architecture-overview)
3. [Technology Stack](#technology-stack)
4. [Data Flow: From Blockchain to Dashboard](#data-flow-from-blockchain-to-dashboard)
5. [Stream Processing with Rustflow](#stream-processing-with-rustflow)
6. [ClickHouse as the Analytics Engine](#clickhouse-as-the-analytics-engine)
7. [Key Design Decisions](#key-design-decisions)
8. [Implementation References](#implementation-references)

---

## The Challenge

Building analytics for Solana DEX trading presents unique challenges:

### Volume & Velocity
- **100,000+ swaps per second** during peak activity
- **15 different timeframes** for OHLCV (1s to 1w)
- **Millions of unique tokens** with varying lifespans (some live hours, others years)
- **Real-time trader position tracking** across every token

### Latency Requirements
- **< 1 second** ingestion lag (blockchain to queryable)
- **< 200ms p95** query latency for API endpoints
- **Real-time** OHLCV updates for TradingView charts

### Complexity
- **Multi-hop swaps** (Jupiter routes through 2-4 pools)
- **Bonding curve tokens** (Pump.fun, Moonshot) with graduation mechanics
- **FIFO-based PnL** calculations requiring trade history
- **Risk metrics** (sniper detection, bundler analysis, whale tracking)

### Scale
- **10,000+ active tokens** at any moment
- **Millions of trader wallets**
- **30+ NATS subjects** for different data streams
- **Petabytes of historical data** with tiered retention

---

## Architecture Overview

Our system consists of three main layers:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           LAYER 1: DATA INGESTION                                │
│                                                                                  │
│   Solana RPC ──► Geyser Plugin ──► Solana Indexer ──► NATS JetStream            │
│                                                                                  │
│   Raw Events: trades, transfers, tokens, pools, liquidity_events                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         LAYER 2: STREAM PROCESSING                               │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    click-datastreams (Rustflow)                          │   │
│   │                                                                          │   │
│   │  Pipelines:                                                              │   │
│   │  ├── global          → Token/Pool metadata storage                       │   │
│   │  ├── token_prices    → Enrichment, OHLCV, TokenMetrics, Discovery       │   │
│   │  ├── click_users     → Positions, Orders                                 │   │
│   │  └── rewards         → Points, Referrals                                 │   │
│   │                                                                          │   │
│   │  State: LMDB (persistent) + DashMap (in-memory cache)                    │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   Output: 40+ NATS subjects with enriched/aggregated data                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          LAYER 3: ANALYTICS ENGINE                               │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         ClickHouse Cluster                               │   │
│   │                                                                          │   │
│   │  NATS Engine Tables (Queue) ──► Materialized Views ──► Storage Tables   │   │
│   │                                                                          │   │
│   │  Table Engines:                                                          │   │
│   │  ├── ReplacingMergeTree  → Latest state (token_metrics, positions)      │   │
│   │  ├── MergeTree           → Time-series (trades, OHLCV, history)         │   │
│   │  ├── AggregatingMergeTree→ Pre-aggregated metrics (trending)            │   │
│   │  └── NATS Engine         → Real-time ingestion from message queue       │   │
│   │                                                                          │   │
│   │  Storage: Hot (NVMe) → Warm (SSD) → Cold (S3)                           │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Stream Processing: Rustflow

Rustflow is our custom Rust-based stream processing framework (similar to Apache Flink). Key features:

| Feature | Description |
|---------|-------------|
| **Stateful Processing** | Two-tier state: DashMap (in-memory) + LMDB (persistent) |
| **Partitioned Pipelines** | Parallel processing by token_mint or trader |
| **Transform Macros** | Declarative transforms with `#[transform]` attributes |
| **Windowed Aggregation** | Tumbling windows for OHLCV across 15 timeframes |
| **At-Least-Once Delivery** | NATS JetStream consumer with ack on success |

### Message Queue: NATS JetStream

Why NATS over Kafka:

| Aspect | NATS JetStream | Kafka |
|--------|----------------|-------|
| **Latency** | ~1ms p99 | ~10ms p99 |
| **Operational Complexity** | Single binary | ZooKeeper + brokers |
| **Memory Footprint** | ~50MB | ~1GB+ |
| **Partitioning** | Subject-based wildcards | Topic partitions |
| **Best For** | Real-time streaming | High durability batch |

We use NATS for its low latency and operational simplicity, accepting the trade-off of weaker durability guarantees (acceptable for re-playable blockchain data).

### Analytics Database: ClickHouse

Why ClickHouse over alternatives:

| Requirement | ClickHouse | TimescaleDB | Druid |
|-------------|------------|-------------|-------|
| **Write Speed** | 1M+ rows/sec | 100K rows/sec | 500K rows/sec |
| **Query Latency** | < 100ms | < 500ms | < 200ms |
| **Compression** | 10-20x | 3-5x | 5-10x |
| **Real-time Ingestion** | Native NATS engine | External | Native |
| **MV Support** | Excellent | Good | Limited |
| **Operational** | Simple | Complex (PG) | Complex |

ClickHouse's native NATS engine eliminates the need for a separate ingestion service.

---

## Data Flow: From Blockchain to Dashboard

### Step 1: Blockchain → NATS (Solana Indexer)

The Solana indexer uses Geyser plugins to capture:

```
Raw NATS Subjects:
├── solana.trades.v2        → Swap events (DEX trades)
├── solana.transfers.v2     → Token transfers
├── solana.tokens           → New token metadata (Metaplex)
├── solana.pools            → Pool/market creation
├── solana.liquidity_events → Add/remove liquidity
├── solana.token_events     → Mint/burn events
└── solana.failed_transactions → Failed swaps
```

### Step 2: NATS → Rustflow → NATS (DataStreams)

DataStreams transforms raw events into analytics-ready data:

```
Input → Transform → Output

solana.trades.v2 ──► enrich_trade ──► solana.enriched_trades
                 ├──► market_ohlcv_* ──► solana.market_ohlcv.{1s,5s,...,1w}
                 ├──► token_ohlcv_* ──► solana.token_ohlcv.{1s,5s,...,1w}
                 ├──► rolling_windows ──► solana.token_metrics.v2
                 └──► discovery_emitter ──► solana.discovery.*

solana.transfers.v2 ──► position_from_transfer ──► solana.position_overview
```

### Step 3: NATS → ClickHouse (NATS Engine)

ClickHouse directly consumes from NATS using its native engine:

```sql
-- NATS Queue table (virtual - just subscription config)
CREATE TABLE enriched_trades_queue (...)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://localhost:4222',
    nats_subjects = 'solana.enriched_trades',
    nats_format = 'RowBinary';

-- Materialized View routes to storage table
CREATE MATERIALIZED VIEW enriched_trades_mv TO enriched_trades AS
SELECT * FROM enriched_trades_queue;

-- Storage table (actual data)
CREATE TABLE enriched_trades (...)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature);
```

---

## Deep Dive: The Complete Data Pipeline

### End-to-End Flow: Blockchain → Dashboard

Let's trace a single trade through the entire system:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE DATA PIPELINE FLOW                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. BLOCKCHAIN EVENT                                                             │
│     └─ User swaps 1 SOL for BONK on Raydium                                     │
│        Transaction: 5K7p9Qk...                                                  │
│                                                                                  │
│  2. INDEXER CAPTURE (Solana Geyser Plugin)                                      │
│     └─ Geyser plugin detects swap instruction                                   │
│     └─ Parses raw amounts, accounts, signatures                                 │
│     └─ Publishes to NATS: solana.trades.v2                                      │
│                                                                                  │
│  3. DATASTREAMS PROCESSING (Rust/Rustflow)                                      │
│     ┌─────────────────────────────────────────────────────────────────────────┐ │
│     │ Pipeline: token_prices                                                   │ │
│     │                                                                          │ │
│     │ Transform: enrich_trade                                                  │ │
│     │ ├─ Read token metadata from state (name, symbol, decimals)              │ │
│     │ ├─ Read pool state (liquidity, reserves)                                │ │
│     │ ├─ Calculate USD values using SOL/USD price                             │ │
│     │ ├─ Detect trade direction (Buy/Sell)                                    │ │
│     │ ├─ Extract fees (tx, priority, dex, platform, validator tip)            │ │
│     │ └─ Emit to: solana.enriched_trades                                      │ │
│     │                                                                          │ │
│     │ Transform: market_ohlcv_5m (windowed)                                    │ │
│     │ ├─ Aggregate trades into 5-min windows by (market, token)               │ │
│     │ ├─ Calculate O/H/L/C from first/max/min/last prices                     │ │
│     │ └─ Emit to: solana.market_ohlcv.5m                                      │ │
│     │                                                                          │ │
│     │ Transform: rolling_window_stats                                          │ │
│     │ ├─ Maintain rolling windows (5m, 1h, 6h, 24h)                           │ │
│     │ ├─ Aggregate volume, price change, buy/sell counts                      │ │
│     │ └─ Emit to: solana.token_metrics.v2                                     │ │
│     └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  4. CLICKHOUSE INGESTION (NATS Engine)                                          │
│     ┌─────────────────────────────────────────────────────────────────────────┐ │
│     │ enriched_trades_queue (NATS Engine)                                      │ │
│     │ └─ Subscribes to: solana.enriched_trades                                │ │
│     │ └─ Deserializes RowBinary into columns                                  │ │
│     │                                                                          │ │
│     │ Materialized View: enriched_trades_mv                                    │ │
│     │ └─ Routes data to: enriched_trades table                                │ │
│     │ └─ Adds: ingested_at timestamp                                          │ │
│     │                                                                          │ │
│     │ Storage Table: enriched_trades (MergeTree)                               │ │
│     │ └─ PARTITION BY toYYYYMM(block_timestamp)                               │ │
│     │ └─ ORDER BY (token_mint, block_timestamp, signature)                    │ │
│     └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  5. API QUERY                                                                    │
│     └─ User requests trade history                                              │
│     └─ Query: SELECT * FROM enriched_trades WHERE token='BONK' LIMIT 100       │
│     └─ Response in < 50ms                                                       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### NATS Subject Architecture

We use a hierarchical subject naming convention:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         NATS SUBJECT HIERARCHY                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  LAYER 1: RAW EVENTS (from Solana Indexer)                                      │
│  ├── solana.trades.v2           Raw swap events                                 │
│  ├── solana.transfers.v2        Token transfers                                 │
│  ├── solana.tokens              New token metadata                              │
│  ├── solana.pools               Pool creation events                            │
│  ├── solana.liquidity_events.v1 Add/remove liquidity                           │
│  ├── solana.token_events.v1     Mint/burn events                               │
│  └── solana.failed_transactions.v1  Failed swaps                               │
│                                                                                  │
│  LAYER 2: ENRICHED DATA (from DataStreams)                                      │
│  ├── solana.enriched_trades     Trades with metadata + fees                    │
│  ├── solana.market_ohlcv.*      Per-pool candles (30 timeframes)               │
│  │   ├── solana.market_ohlcv.1s                                                │
│  │   ├── solana.market_ohlcv.5m                                                │
│  │   └── ... (15 timeframes)                                                   │
│  ├── solana.token_ohlcv.*       Per-token candles (30 timeframes)              │
│  ├── solana.token_metrics.v2    89-field token metrics                         │
│  ├── solana.position_overview   Trader positions                               │
│  └── solana.discovery.*         New/graduating token alerts                    │
│      ├── solana.discovery.new                                                  │
│      ├── solana.discovery.graduating                                           │
│      └── solana.discovery.graduated                                            │
│                                                                                  │
│  LAYER 3: PLATFORM-SPECIFIC (from DataStreams)                                  │
│  ├── click.order_overview       Platform order tracking                        │
│  ├── click.user_rewards         Points and rewards                             │
│  └── click.referral_earnings    Referral program                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### RowBinary Serialization: The Performance Secret

We use ClickHouse's native RowBinary format for all NATS messages:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    JSON vs RowBinary COMPARISON                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  SAME TRADE DATA:                                                               │
│                                                                                  │
│  JSON (Human readable, ~520 bytes):                                             │
│  {                                                                              │
│    "signature": "5K7p9QkABC123...",                                            │
│    "trader": "7xKXtg2CED456...",                                               │
│    "token_mint": "DezXAZ8z7Pnr...",                                            │
│    "volume_sol": "1.500000000000000000",                                       │
│    "volume_usd": "225.000000000000000000",                                     │
│    ...                                                                          │
│  }                                                                              │
│                                                                                  │
│  RowBinary (Machine optimized, ~150 bytes):                                     │
│  [length-prefixed strings + fixed-width decimals + timestamps]                  │
│  5B 35 4B 37 70 39 51 6B 41 42 43 31 32 33 ...                                  │
│                                                                                  │
│  BENEFITS:                                                                       │
│  ├── 3.5x smaller message size                                                  │
│  ├── Zero parsing overhead (direct memory mapping)                              │
│  ├── Native ClickHouse format (no conversion)                                   │
│  └── Result: 3x higher throughput                                               │
│                                                                                  │
│  RUST IMPLEMENTATION:                                                           │
│  We use the `klickhouse` crate with #[derive(Row)] macro                        │
│  to auto-generate RowBinary serialization                                       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse NATS Engine: Direct Ingestion

ClickHouse can subscribe directly to NATS subjects:

```sql
-- Step 1: Create NATS queue table (virtual - just subscription config)
CREATE TABLE enriched_trades_queue (
    signature String,
    trader String,
    token_mint String,
    volume_sol Decimal128(18),
    -- ... all 45 fields in exact struct order
)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats:4222',
    nats_subjects = 'solana.enriched_trades',
    nats_format = 'RowBinary',
    nats_num_consumers = 4;  -- Parallel consumer threads

-- Step 2: Materialized View routes to storage
CREATE MATERIALIZED VIEW enriched_trades_mv TO enriched_trades AS
SELECT
    *,
    now64(3) AS ingested_at  -- Add ClickHouse timestamp
FROM enriched_trades_queue;

-- Step 3: Storage table with optimizations
CREATE TABLE enriched_trades (
    -- All fields + indexes + compression codecs
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature);
```

### Fan-Out Pattern: One Queue, Multiple Destinations

A single NATS queue can feed multiple tables for different query patterns:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         FAN-OUT PATTERN                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                    enriched_trades_queue                                        │
│                           │                                                      │
│           ┌───────────────┼───────────────┬───────────────┐                     │
│           │               │               │               │                     │
│           ▼               ▼               ▼               ▼                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ enriched_   │  │ trades_by_  │  │ trades_by_  │  │ trades_     │            │
│  │ trades      │  │ trader      │  │ market      │  │ hourly_agg  │            │
│  │             │  │             │  │             │  │             │            │
│  │ ORDER BY:   │  │ ORDER BY:   │  │ ORDER BY:   │  │ GROUP BY:   │            │
│  │ (token,time)│  │ (trader,    │  │ (market,    │  │ (hour,      │            │
│  │             │  │  time)      │  │  time)      │  │  token,dex) │            │
│  │             │  │             │  │             │  │             │            │
│  │ Query:      │  │ Query:      │  │ Query:      │  │ Query:      │            │
│  │ Token trades│  │ Wallet hist │  │ Pool trades │  │ Aggregations│            │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                                  │
│  BENEFITS:                                                                       │
│  ├── Each table optimized for specific query pattern                            │
│  ├── Different TTL policies per table                                           │
│  ├── Independent backpressure handling                                          │
│  └── No duplication in NATS (single subscription)                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Stream Processing with Rustflow

### Transform Anatomy

A Rustflow transform is a pure function decorated with metadata:

```rust
#[transform(
    pipeline = "token_prices",           // Pipeline shard
    name = "enrich_trade",               // Unique name
    source = "solana.trades.v2",         // Input NATS subject
    partition_by = "token_mint",         // Partitioning key
    serializer = RowBinary              // Output format
)]
fn enrich_trade(row: Row<Swap>, context: Context) -> Result<Trade, Error> {
    // Access state (in-memory + persistent)
    let token = context.state.get::<Token>(&swap.token_mint)?;
    let pool = context.state.get::<Pool>(&swap.market)?;

    // Enrich the swap with metadata
    let trade = Trade {
        signature: swap.signature,
        token_mint: swap.token_mint,
        token_name: token.name,
        token_symbol: token.symbol,
        // ... enriched fields
    };

    // Emit metrics
    counter!("trades_enriched").increment(1);

    Ok(trade)
}
```

### State Management

Rustflow uses a two-tier state architecture:

```
┌────────────────────────────────────────────────────────┐
│                    Application                          │
│                                                         │
│   context.state.get(&key)  context.state.set(&key, v)  │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                   DashMap (In-Memory)                   │
│                                                         │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐               │
│   │ Token A │  │ Pool X  │  │Position │   O(1) reads  │
│   └─────────┘  └─────────┘  └─────────┘               │
└────────────────────────────────────────────────────────┘
                           │
                    Async batch write
                           ▼
┌────────────────────────────────────────────────────────┐
│                    LMDB (Persistent)                    │
│                                                         │
│   ./state/token_prices.lmdb                            │
│   - Crash recovery                                      │
│   - Checkpoint on graceful shutdown                     │
└────────────────────────────────────────────────────────┘
```

### State Types

All state types derive from `AnyState`:

```rust
use rustflow::AnyState;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, AnyState)]
pub struct TokenState {
    pub mint: String,
    pub name: String,
    pub symbol: String,
    pub decimals: u8,
    pub creator: String,
}
```

### Windowed Aggregation

OHLCV aggregation uses tumbling windows:

```rust
#[transform(
    pipeline = "token_prices",
    name = "token_ohlcv.1m",
    source = "solana.enriched_trades",
    partition_by = "token_mint",
    window = "1m",                      // 1-minute tumbling window
    serializer = RowBinary
)]
fn token_ohlcv_1m(window: Window<Trade>, context: Context) -> Result<Ohlcv, Error> {
    let trades = window.items();

    Ok(Ohlcv {
        token: trades[0].token_mint.clone(),
        o_price_sol: trades.first().unwrap().spot_price_sol,
        h_price_sol: trades.iter().map(|t| t.spot_price_sol).max().unwrap(),
        l_price_sol: trades.iter().map(|t| t.spot_price_sol).min().unwrap(),
        c_price_sol: trades.last().unwrap().spot_price_sol,
        v_sol: trades.iter().map(|t| t.volume_sol).sum(),
        buys: trades.iter().filter(|t| t.trade_direction == Buy).count() as u64,
        sells: trades.iter().filter(|t| t.trade_direction == Sell).count() as u64,
        timestamp: window.start_time(),
    })
}
```

---

## ClickHouse as the Analytics Engine

### Table Engine Selection

| Engine | Use Case | Example Table |
|--------|----------|---------------|
| **MergeTree** | Append-only time-series | `enriched_trades`, OHLCV history |
| **ReplacingMergeTree** | Latest state per key | `token_metrics`, `position_overview` |
| **AggregatingMergeTree** | Pre-computed aggregates | `trending_volume_daily` |
| **NATS Engine** | Real-time queue ingestion | `*_queue` tables |

### Partitioning Strategy

```sql
-- Time-series: Partition by month
PARTITION BY toYYYYMM(block_timestamp)

-- High-frequency: Partition by day (for efficient TTL)
PARTITION BY toYYYYMMDD(candle_time)

-- State tables: No partitioning (ReplacingMergeTree)
-- ORDER BY defines the primary key
```

### Tiered TTL Strategy

Different data has different retention needs:

| Data Type | Hot (NVMe) | Warm (SSD) | Cold (S3) | Delete |
|-----------|------------|------------|-----------|--------|
| 1s OHLCV | 1 day | - | - | 7 days |
| 1m OHLCV | 7 days | 30 days | - | 90 days |
| 1d OHLCV | 30 days | 90 days | Forever | Never |
| Trades | 7 days | 30 days | 365 days | Never |
| Token Metrics | Forever | - | - | Never |

```sql
-- Example: Tiered OHLCV table
CREATE TABLE token_ohlcv_medium (...)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(candle_time)
ORDER BY (token, timeframe, candle_time)
TTL candle_time + INTERVAL 7 DAY TO VOLUME 'warm',
    candle_time + INTERVAL 90 DAY DELETE
SETTINGS
    index_granularity = 8192,
    ttl_only_drop_parts = 1;  -- Efficient part-level TTL
```

---

## Key Design Decisions

### 1. RowBinary Serialization

We use ClickHouse's native RowBinary format for NATS messages:

**Why not JSON?**
- JSON: ~500 bytes per trade, parsing overhead
- RowBinary: ~150 bytes per trade, zero-copy parsing
- **3x throughput improvement**

```rust
// Rust: klickhouse crate for RowBinary
#[derive(Row, Serialize, Deserialize)]
pub struct Trade {
    pub signature: String,
    pub volume_sol: Decimal,
    // ... fields in ClickHouse column order
}
```

### 2. Fan-Out Pattern for Materialized Views

One NATS queue feeds multiple storage tables:

```
enriched_trades_queue ──┬──► enriched_trades (main history)
                        ├──► enriched_trades_by_trader (index)
                        ├──► enriched_trades_by_market (index)
                        └──► enriched_trades_hourly_agg (aggregation)
```

Each MV is a separate consumer, enabling:
- Independent backpressure
- Different ORDER BY for query patterns
- Different TTL policies

### 3. Deduplication Strategy

At-least-once delivery means we handle duplicates:

```sql
-- ReplacingMergeTree: Keep latest version
CREATE TABLE token_metrics (
    token_mint String,
    price_usd Decimal(38, 18),
    last_updated DateTime64(9),
    version UInt64 MATERIALIZED toUnixTimestamp64Nano(last_updated)
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (token_mint);

-- Query with FINAL to deduplicate
SELECT * FROM token_metrics FINAL WHERE token_mint = '...';
```

### 4. Separate Tables by Timeframe Tier

Rather than one table with conditional TTL:

```sql
-- DON'T: Mixed TTL in one table
TTL candle_time + toIntervalDay(multiIf(timeframe = '1s', 7, ...))

-- DO: Separate tables by retention tier
market_ohlcv_short  → 1s, 5s, 15s, 30s  (7 day TTL)
market_ohlcv_medium → 1m, 5m, 15m, 1h   (90 day TTL)
market_ohlcv_long   → 4h, 1d, 1w        (Forever)
```

Benefits:
- `ttl_only_drop_parts = 1` works (parts contain same-TTL data)
- No rewrite overhead from row-level deletion
- Targeted queries go to specific tier

---

## Implementation References

This blog series references our actual implementation. Key documentation:

| Document | Content |
|----------|---------|
| [17_DATA_FLOW.md](./17_DATA_FLOW.md) | Complete data flow architecture |
| [08_ENRICHED_TRADES_SETUP.md](./08_ENRICHED_TRADES_SETUP.md) | Trade enrichment & DDL |
| [10_OHLCV_SETUP.md](./10_OHLCV_SETUP.md) | OHLCV tiered storage |
| [09_TOKEN_METRICS_SETUP.md](./09_TOKEN_METRICS_SETUP.md) | Rolling window metrics |
| [11_POSITION_OVERVIEW_SETUP.md](./11_POSITION_OVERVIEW_SETUP.md) | Position tracking |
| [31_PRD_CLICKHOUSE_MAPPING.md](./31_PRD_CLICKHOUSE_MAPPING.md) | PRD to implementation mapping |

---

## What's Next

In **Part 2: Trade Enrichment & OHLCV Generation**, we'll dive deep into:

1. How we enrich raw swaps with token metadata
2. OHLCV aggregation across 15 timeframes
3. Handling multi-hop swaps (Jupiter routes)
4. Fee decomposition (transaction, priority, validator tips, DEX, platform)
5. ClickHouse table design for trade analytics

---

## Metrics We Track

After implementing this architecture, we achieved:

| Metric | Value |
|--------|-------|
| **Ingestion Lag** | p95 < 800ms |
| **Query Latency** | p95 < 150ms |
| **Trades/Second** | 100,000+ sustained |
| **OHLCV Subjects** | 30 (15 market + 15 token) |
| **Storage Compression** | 12:1 average |
| **State Recovery** | < 30 seconds |

---

*Part 1 of 6 in the "Building Real-Time Solana DEX Analytics with ClickHouse" series.*

*For implementation details, see our [ClickHouse Setup Documentation](./02_CLICKHOUSE_SETUP_INDEX.md).*
