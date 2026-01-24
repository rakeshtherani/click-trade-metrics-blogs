# Click Trade ClickHouse Technical Reference

## Executive Summary

This is the definitive technical reference for Click Trade's ClickHouse implementation. It documents all 70+ optimization patterns from production blockchain analytics systems (CryptoHouse, StockHouse) and maps them directly to our `datastreams` database schema.

**What This Document Covers:**
- Complete schema design rationale with "why" explanations
- All optimization patterns with before/after comparisons
- Performance benchmarks and expected latencies
- Anti-patterns and common mistakes to avoid
- Operational runbooks and troubleshooting guides
- Capacity planning and cost analysis

**Target Audience:** Backend engineers, DevOps, Data engineers

---

# Part 1: Architecture Overview

## 1.1 Cluster Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ClickHouse Cluster: server-apex                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   Replica 0-0   │  │   Replica 0-1   │  │   Replica 0-2   │              │
│  │   (Primary)     │  │   (Secondary)   │  │   (Secondary)   │              │
│  │                 │  │                 │  │                 │              │
│  │  NATS Queues    │  │  Read Replicas  │  │  Read Replicas  │              │
│  │  MVs            │  │                 │  │                 │              │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                    │                        │
│           └────────────────────┼────────────────────┘                        │
│                                │                                             │
│                    ┌───────────┴───────────┐                                │
│                    │     ZooKeeper         │                                │
│                    │  (Replication Mgmt)   │                                │
│                    └───────────────────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Configuration:**
| Setting | Value | Why |
|---------|-------|-----|
| Cluster Name | `server-apex` | Matches existing infrastructure |
| Shards | 1 | Single shard for now (can scale later) |
| Replicas | 3 | High availability, read scaling |
| ZooKeeper Path | `/clickhouse/tables/datastreams/{table}/{shard}` | Standard path pattern |

**Why Single Shard?**
- Current data volume (~100K trades/day) fits single shard
- Simplifies queries (no distributed joins)
- Can add shards later when needed (horizontal scaling)
- 3 replicas provide HA and read scaling

---

## 1.2 Data Flow Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                    DATA FLOW                                              │
└──────────────────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────┐
    │  Solana     │
    │  Blockchain │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐     ┌─────────────┐
    │ DataStreams │────▶│    NATS     │
    │   (Rust)    │     │   Server    │
    └─────────────┘     └──────┬──────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │ enriched_   │     │ token_      │     │ position_   │
    │ trades_queue│     │ metrics_q   │     │ overview_q  │
    │ (NATS Eng)  │     │ (NATS Eng)  │     │ (NATS Eng)  │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │ MV: route   │     │ MV: route   │     │ MV: route   │
    │ to tables   │     │ to table    │     │ to table    │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
     ┌─────┴─────┐             │                   │
     │           │             │                   │
     ▼           ▼             ▼                   ▼
┌─────────┐ ┌─────────┐  ┌─────────┐         ┌─────────┐
│enriched │ │enriched │  │ token_  │         │position │
│_trades  │ │_trades_ │  │ metrics │         │_overview│
│         │ │by_trader│  │         │         │         │
└────┬────┘ └─────────┘  └─────────┘         └─────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│              Aggregation Materialized Views              │
├─────────────┬─────────────┬─────────────┬───────────────┤
│ OHLCV MVs   │ Trending MVs│ Discovery   │ DLQ           │
│ (1m,5m,15m, │ (daily agg) │ Updates     │ (errors)      │
│  1h,4h,1d)  │             │             │               │
└─────────────┴─────────────┴─────────────┴───────────────┘
```

**Why This Architecture?**

| Component | Purpose | Alternative Considered | Why Rejected |
|-----------|---------|----------------------|--------------|
| NATS Engine | Real-time ingestion | Kafka Engine | NATS simpler, lower latency, already in stack |
| Null + MV | Staging pattern | Direct insert | Can't transform/validate without staging |
| Secondary tables | Query optimization | Single table + projections | Projections have limitations on aggregates |
| Aggregating MVs | Pre-computed OHLCV | Query-time aggregation | Too slow for real-time dashboards |

---

## 1.3 Table Inventory

### Primary Tables (Current State)

| Table | Engine | Rows (Est) | Purpose |
|-------|--------|------------|---------|
| `token_metrics` | ReplacingMergeTree | ~50K | Current token state |
| `position_overview` | ReplacingMergeTree | ~500K | Current positions |
| `discovery` | ReplacingMergeTree | ~50K | Token discovery state |
| `user_orders` | ReplacingMergeTree | ~100K | Order state |

### Event Tables (Append-Only)

| Table | Engine | Rows/Day (Est) | Retention |
|-------|--------|----------------|-----------|
| `enriched_trades` | MergeTree | ~100K | Forever |
| `token_metrics_history` | MergeTree | ~500K | 6 months hot, then cold |
| `position_history` | MergeTree | ~100K | 6 months hot, then cold |
| `discovery_history` | MergeTree | ~50K | 6 months hot, then cold |
| `dlq_failed_records` | MergeTree | ~1K | 30 days (DELETE) |

### Aggregation Tables

| Table | Engine | Granularity | Retention |
|-------|--------|-------------|-----------|
| `market_ohlcv_1m` | AggregatingMergeTree | 1 minute | 7 days |
| `market_ohlcv_5m` | AggregatingMergeTree | 5 minutes | 30 days |
| `market_ohlcv_15m` | AggregatingMergeTree | 15 minutes | 90 days |
| `market_ohlcv_1h` | AggregatingMergeTree | 1 hour | 1 year |
| `market_ohlcv_4h` | AggregatingMergeTree | 4 hours | Forever |
| `market_ohlcv_1d` | AggregatingMergeTree | 1 day | Forever |
| `trending_*` | AggregatingMergeTree | Daily | 1 year |

---

# Part 2: Schema Design Deep Dive

## 2.1 Data Type Selection

### Financial Precision: Why Decimal(38, 18)?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FINANCIAL PRECISION DECISION TREE                         │
└─────────────────────────────────────────────────────────────────────────────┘

Is it a financial amount (price, volume, balance)?
│
├── YES ──▶ Use Decimal(38, 18)
│           │
│           └── Why 18 decimals?
│               • Ethereum standard = 18 decimals
│               • Solana (lamports) = 9 decimals
│               • 18 covers both + future tokens
│               • 38 total digits = handles $1 quadrillion
│
└── NO ──▶ Is it a count or sequence?
           │
           ├── YES ──▶ Use UInt64
           │
           └── NO ──▶ Is it a timestamp?
                      │
                      ├── Block time ──▶ DateTime64(9) (nanoseconds)
                      ├── Server time ──▶ DateTime64(3) (milliseconds)
                      └── Unix epoch ──▶ Int64 (for ATH timestamps)
```

**The Float64 Mistake (Don't Do This!):**

```sql
-- ❌ WRONG: Float loses precision
CREATE TABLE bad_trades (
    price Float64,    -- 0.1 + 0.2 = 0.30000000000000004
    volume Float64
);

-- ✅ CORRECT: Decimal preserves exact values
CREATE TABLE good_trades (
    price Decimal(38, 18),   -- 0.1 + 0.2 = 0.3 exactly
    volume Decimal(38, 18)
);
```

**Real-World Impact:**
```
User buys 1000 tokens at 0.001 USD each
Float64:  1000 * 0.001 = 0.9999999999999999 (displays as $1.00 but isn't)
Decimal:  1000 * 0.001 = 1.000000000000000000 (exactly $1.00)

Over millions of trades, Float errors compound into significant discrepancies.
```

### String Type Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STRING TYPE DECISION MATRIX                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              How many unique values?
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
       < 256                      256 - 10,000                  > 10,000
          │                            │                            │
          ▼                            ▼                            ▼
    ┌───────────┐              ┌───────────────┐              ┌───────────┐
    │  Enum8    │              │LowCardinality │              │  String   │
    │           │              │   (String)    │              │           │
    └───────────┘              └───────────────┘              └───────────┘
          │                            │                            │
    Examples:                    Examples:                    Examples:
    • trade_direction           • token_symbol               • trader address
      (BUY/SELL)                • dex (~20)                  • token_mint
    • order_status              • venue (~10)                • signature
      (7 values)                • stage (~5)                 • tx_hash
    • side (2 values)           • error_type (~20)
```

**Why LowCardinality Matters:**

```
Without LowCardinality:
┌─────────────┐
│   "Raydium" │  7 bytes × 1M rows = 7 MB
│   "Raydium" │
│   "Raydium" │
│   ... ×1M   │
└─────────────┘

With LowCardinality:
┌─────────────┐     ┌──────────────────┐
│ Dictionary  │     │  Data Column     │
├─────────────┤     ├──────────────────┤
│ 0: "Raydium"│     │ 0, 0, 0, 0, ...  │  1 byte × 1M = 1 MB
│ 1: "Jupiter"│     │ (just indices)   │
│ 2: "Orca"   │     │                  │
└─────────────┘     └──────────────────┘

Storage: 7 MB → 1 MB (7x compression)
GROUP BY: 7x faster (comparing integers, not strings)
```

---

## 2.2 Codec Strategy

### Codec Selection Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CODEC DECISION TREE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

What type of data?
│
├── Timestamps ──▶ DoubleDelta, ZSTD(1)
│   │
│   └── Why? Timestamps are sequential. DoubleDelta stores
│            difference of differences (often 0 or 1).
│            1 second intervals: 8 bytes → 0.1 bytes (80x compression)
│
├── Monotonic integers (slot, sequence) ──▶ Delta, ZSTD(1)
│   │
│   └── Why? Values always increase. Delta stores differences.
│            Slot 1000, 1001, 1002 → stored as 1000, 1, 1
│
├── Other integers ──▶ T64, ZSTD(1)
│   │
│   └── Why? T64 transposes 64 values, groups similar bits.
│            Good for integers that aren't sequential.
│
├── High-cardinality strings ──▶ ZSTD(3)
│   │
│   └── Why? ZSTD(3) = higher compression, slightly slower.
│            Worth it for large strings (addresses = 44 chars).
│            ZSTD(3) vs ZSTD(1): 20% smaller, 10% slower.
│
├── Decimals ──▶ ZSTD(1)
│   │
│   └── Why? Prices have patterns (many end in .00, .50).
│            ZSTD catches these patterns well.
│
└── Dedup hash ──▶ Delta, ZSTD(1)
    │
    └── Why? Hash values from same time window are often similar.
             Delta + ZSTD handles this well.
```

### Applied Codecs in enriched_trades

```sql
CREATE TABLE datastreams.enriched_trades
(
    -- ZSTD(3) for high-cardinality strings (5:1 compression)
    signature String CODEC(ZSTD(3)),
    trader String CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),
    market String CODEC(ZSTD(3)),

    -- Delta for monotonic slot (8:1 compression)
    slot UInt64 CODEC(Delta, ZSTD(1)),

    -- DoubleDelta for sequential timestamps (16:1 compression)
    block_timestamp DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),
    server_timestamp DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),

    -- ZSTD(1) for decimals (4:1 compression)
    volume_sol Decimal128(18) CODEC(ZSTD(1)),
    volume_usd Decimal128(18) CODEC(ZSTD(1)),
    execution_price_sol Decimal128(18) CODEC(ZSTD(1)),
    execution_price_usd Decimal128(18) CODEC(ZSTD(1)),

    -- T64 for non-monotonic integers
    hop_index UInt8 CODEC(T64, ZSTD(1)),

    -- Delta for dedup hash (similar values in time window)
    dedup_hash UInt64 MATERIALIZED cityHash64(signature, toString(hop_index))
        CODEC(Delta, ZSTD(1)),

    -- LowCardinality handles its own compression (10:1)
    dex LowCardinality(String),
    token_symbol LowCardinality(String),

    -- Enum8 = 1 byte, no codec needed
    trade_direction Enum8('BUY' = 0, 'SELL' = 1)
)
```

### Compression Benchmarks

| Column | Raw Size | Compressed | Ratio | Notes |
|--------|----------|------------|-------|-------|
| block_timestamp | 8 bytes | 0.5 bytes | 16:1 | DoubleDelta magic |
| trader (address) | 44 bytes | 8 bytes | 5.5:1 | ZSTD(3) |
| slot | 8 bytes | 1 byte | 8:1 | Delta |
| volume_sol | 16 bytes | 4 bytes | 4:1 | ZSTD(1) |
| dex | ~8 bytes | 0.5 bytes | 16:1 | LowCardinality |
| trade_direction | 1 byte | 1 byte | 1:1 | Enum8 already optimal |

**Total per trade row:** ~200 bytes raw → ~30 bytes compressed (6.7x overall)

---

## 2.3 Table Engine Selection

### Engine Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       TABLE ENGINE DECISION TREE                             │
└─────────────────────────────────────────────────────────────────────────────┘

Does this table store "current state" that gets updated?
│
├── YES ──▶ ReplacingMergeTree
│   │       (keeps latest version by ORDER BY key)
│   │
│   │       Examples:
│   │       • token_metrics (latest metrics per token)
│   │       • position_overview (latest position per trader-token)
│   │       • discovery (latest state per token)
│   │       • user_orders (latest order state)
│   │
│   └── Do you need a version column?
│       │
│       ├── YES (explicit ordering) ──▶ ReplacingMergeTree(version_column)
│       │   • version = last_updated timestamp
│       │   • Ensures newest wins even if inserted out of order
│       │
│       └── NO (insert order is correct) ──▶ ReplacingMergeTree()
│           • Keeps whichever was inserted last
│
└── NO (append-only) ──▶ Do you need pre-aggregation?
                         │
    ┌────────────────────┴────────────────────┐
    │                                         │
   YES                                        NO
    │                                         │
    ▼                                         ▼
Need unique counts (HyperLogLog)?        MergeTree
    │                                    (simple append)
    ├── YES ──▶ AggregatingMergeTree
    │   • Uses AggregateFunction columns  Examples:
    │   • uniqCombined(14) for unique counts  • enriched_trades
    │   • argMin/argMax for OHLC              • *_history tables
    │                                         • dlq_failed_records
    │   Examples:
    │   • market_ohlcv_* (OHLC + unique traders)
    │   • trending_* (unique tokens, traders)
    │
    └── NO (just sums/counts) ──▶ Consider SummingMergeTree
        • Simpler than AggregatingMergeTree
        • Auto-sums numeric columns on merge
        • BUT: Can't do unique counts accurately
```

### Why ReplacingMergeTree for State Tables?

**The Problem:**
```
Incoming events for same token:
t=0: token_mint=ABC, price=1.00, holders=100
t=1: token_mint=ABC, price=1.05, holders=102
t=2: token_mint=ABC, price=1.03, holders=101

Without ReplacingMergeTree (regular MergeTree):
┌────────────┬───────┬─────────┐
│ token_mint │ price │ holders │
├────────────┼───────┼─────────┤
│ ABC        │ 1.00  │ 100     │  ← Old data still here!
│ ABC        │ 1.05  │ 102     │  ← Old data still here!
│ ABC        │ 1.03  │ 101     │  ← Current
└────────────┴───────┴─────────┘

Query: SELECT * FROM token_metrics WHERE token_mint = 'ABC'
Returns: 3 rows! Which is correct?
```

**The Solution:**
```sql
ENGINE = ReplacingMergeTree(last_updated)
ORDER BY (token_mint)

After background merge:
┌────────────┬───────┬─────────┐
│ token_mint │ price │ holders │
├────────────┼───────┼─────────┤
│ ABC        │ 1.03  │ 101     │  ← Only latest kept!
└────────────┴───────┴─────────┘

Query: SELECT * FROM token_metrics FINAL WHERE token_mint = 'ABC'
Returns: 1 row (the latest)
```

**Important:** Always use `FINAL` in queries or handle deduplication in application!

---

## 2.4 ORDER BY Strategy

### ORDER BY Best Practices

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ORDER BY DESIGN RULES                                │
└─────────────────────────────────────────────────────────────────────────────┘

Rule 1: Most filtered column FIRST
─────────────────────────────────────
If 90% of queries filter by token_mint:
ORDER BY (token_mint, block_timestamp, ...)

Rule 2: Low cardinality before high cardinality (within filter group)
─────────────────────────────────────────────────────────────────────
If you filter by both dex (20 values) and token_mint (50K values):
ORDER BY (dex, token_mint, ...)  ← dex first (fewer unique values)

Rule 3: Include uniqueness column last
─────────────────────────────────────────
ORDER BY (token_mint, block_timestamp, signature)
                                   └── Makes each row unique

Rule 4: Time-bucketed ORDER BY for high-frequency data
─────────────────────────────────────────────────────────
For millions of trades per day:
ORDER BY (token_mint, toStartOfHour(block_timestamp), signature)
                      └── Groups data into hour buckets on disk
```

### Click Trade ORDER BY Mapping

| Table | ORDER BY | Query Pattern |
|-------|----------|---------------|
| `enriched_trades` | `(token_mint, block_timestamp, signature)` | "Show trades for token X" |
| `enriched_trades_by_trader` | `(trader, token_mint, block_timestamp)` | "Show portfolio for trader Y" |
| `enriched_trades_by_market` | `(market, block_timestamp)` | "Show trades on market Z" |
| `token_metrics` | `(token_mint)` | "Get current state of token X" |
| `position_overview` | `(trader, token_mint)` | "Get position for trader Y on token X" |
| `market_ohlcv_*` | `(market, candle_time)` | "Get candles for market Z from time A to B" |
| `trending_*` | `(date, venue)` | "Get trending data for date D on venue V" |

### Using PROJECTIONS for Alternative Query Patterns

```sql
-- enriched_trades: Primary ORDER BY optimizes token lookups
-- But we also need efficient trader and dex lookups

CREATE TABLE datastreams.enriched_trades
(
    -- columns...
)
ENGINE = ReplicatedMergeTree(...)
ORDER BY (token_mint, block_timestamp, signature)

-- PROJECTION for trader queries (automatic index)
PROJECTION proj_by_trader
(
    SELECT *
    ORDER BY (trader, token_mint, block_timestamp)
)

-- PROJECTION for dex queries
PROJECTION proj_by_dex
(
    SELECT *
    ORDER BY (dex, block_timestamp, token_mint)
)
```

**How Projections Work:**

```
Query: SELECT * FROM enriched_trades WHERE trader = 'ABC'

Without projection:
├── Scan entire table
├── Filter by trader
└── Slow! O(n) where n = all trades

With projection:
├── Query planner sees proj_by_trader
├── Uses projection's ORDER BY index
└── Fast! O(log n) lookup + sequential read
```

---

## 2.5 Partitioning Strategy

### Partition Alignment with TTL

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PARTITION + TTL ALIGNMENT RULES                           │
└─────────────────────────────────────────────────────────────────────────────┘

TTL = 7 days   → PARTITION BY toYYYYMMDD() (daily)
TTL = 30 days  → PARTITION BY toYYYYMMDD() (daily)
TTL = 90 days  → PARTITION BY toYYYYMMDD() (daily) or toYYYYMM() (monthly)
TTL = 1 year   → PARTITION BY toYYYYMM() (monthly)
TTL = forever  → PARTITION BY toYYYYMM() (monthly) or no partitioning

WHY alignment matters:
─────────────────────────
With ttl_only_drop_parts = 1:
• ClickHouse drops ENTIRE partitions when all rows expire
• If partition = month but TTL = 7 days:
  - Can't drop partition until ALL rows in month expire
  - 23+ days of expired data still on disk!

• If partition = day and TTL = 7 days:
  - Each day's partition drops as soon as it's 7 days old
  - Optimal disk usage
```

### Click Trade Partition Scheme

| Table | Partition | TTL | Alignment |
|-------|-----------|-----|-----------|
| `enriched_trades` | `toYYYYMMDD(block_timestamp)` | None | N/A |
| `token_metrics` | `toYYYYMMDD(last_updated)` | None | N/A |
| `token_metrics_history` | `toYYYYMMDD(last_updated)` | 6 months → cold | Daily for fast TTL |
| `position_history` | `toYYYYMMDD(last_updated)` | 6 months → cold | Daily for fast TTL |
| `market_ohlcv_1m` | `toYYYYMMDD(candle_time)` | 7 days DELETE | Perfect alignment |
| `market_ohlcv_5m` | `toYYYYMMDD(candle_time)` | 30 days DELETE | Good alignment |
| `market_ohlcv_1h` | `toYYYYMMDD(candle_time)` | 1 year DELETE | Acceptable |
| `market_ohlcv_1d` | `toYYYYMMDD(candle_time)` | None | N/A |
| `dlq_failed_records` | `toYYYYMMDD(failed_at)` | 30 days DELETE | Perfect alignment |
| `trending_*` | `toYYYYMMDD(date)` | 1 year DELETE | Good alignment |

---

## 2.6 Tiered Storage (Hot/Cold)

### Storage Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TIERED STORAGE ARCHITECTURE                         │
└─────────────────────────────────────────────────────────────────────────────┘

                    HOT VOLUME                         COLD VOLUME
              ┌─────────────────────┐           ┌─────────────────────┐
              │    Local NVMe SSD   │           │    S3/Object Store  │
              │                     │           │                     │
              │  • < 6 months data  │   TTL     │  • > 6 months data  │
              │  • Fast queries     │  ─────▶   │  • Archived queries │
              │  • ~$0.10/GB/month  │           │  • ~$0.02/GB/month  │
              │                     │           │                     │
              └─────────────────────┘           └─────────────────────┘

Tables using tiered storage:
• token_metrics_history
• position_history
• discovery_history
• user_orders_history
```

### TTL Configuration

```sql
-- token_metrics_history: Move to cold after 6 months
CREATE TABLE datastreams.token_metrics_history
(
    -- columns...
)
ENGINE = ReplicatedMergeTree(...)
ORDER BY (token_mint, last_updated)
PARTITION BY toYYYYMMDD(last_updated)
TTL toDateTime(last_updated) + INTERVAL 6 MONTH TO VOLUME 'cold_volume'
SETTINGS
    storage_policy = 'tiered_storage',
    ttl_only_drop_parts = 1;
```

### Cost Analysis

| Storage Type | Cost/GB/Month | 1TB/Month | 10TB/Month |
|--------------|---------------|-----------|------------|
| NVMe SSD (hot) | $0.10 | $100 | $1,000 |
| S3 Standard (cold) | $0.023 | $23 | $230 |
| S3 Glacier (archive) | $0.004 | $4 | $40 |

**With tiered storage:**
- 2TB hot (recent) + 10TB cold (historical) = $200 + $230 = $430/month
- Without tiering: 12TB all on SSD = $1,200/month
- **Savings: 64%**

---

# Part 3: Aggregation Patterns (OHLCV & Trending)

## 3.1 OHLCV with argMinState/argMaxState

### The Problem with Simple Aggregation

```sql
-- ❌ WRONG: This gives WRONG open/close prices!
SELECT
    toStartOfMinute(block_timestamp) AS candle_time,
    min(price) AS open,   -- This is the LOWEST price, not FIRST!
    max(price) AS high,
    min(price) AS low,
    max(price) AS close   -- This is the HIGHEST price, not LAST!
FROM trades
GROUP BY candle_time;
```

### The Correct Pattern

```sql
-- ✅ CORRECT: argMin/argMax gets price at first/last timestamp
SELECT
    toStartOfMinute(block_timestamp) AS candle_time,
    argMin(price, block_timestamp) AS open,   -- Price at EARLIEST timestamp
    max(price) AS high,
    min(price) AS low,
    argMax(price, block_timestamp) AS close   -- Price at LATEST timestamp
FROM trades
GROUP BY candle_time;
```

### Storing as AggregateFunction

```sql
-- For AggregatingMergeTree, store as STATE
CREATE TABLE datastreams.market_ohlcv_1m
(
    market String,
    candle_time DateTime64(9),

    -- OHLC: Store aggregate STATE, not final value
    open_price AggregateFunction(argMin, Decimal128(18), DateTime64(9)),
    close_price AggregateFunction(argMax, Decimal128(18), DateTime64(9)),

    -- High/Low: SimpleAggregateFunction is simpler for min/max
    high_price SimpleAggregateFunction(max, Decimal128(18)),
    low_price SimpleAggregateFunction(min, Decimal128(18)),

    -- Volume and counts
    volume SimpleAggregateFunction(sum, Decimal128(18)),
    buy_volume SimpleAggregateFunction(sum, Decimal128(18)),
    sell_volume SimpleAggregateFunction(sum, Decimal128(18)),
    trade_count SimpleAggregateFunction(sum, UInt64),

    -- Unique traders (HyperLogLog)
    unique_traders AggregateFunction(uniqCombined(14), String)
)
ENGINE = ReplicatedAggregatingMergeTree(...)
ORDER BY (market, candle_time)
PARTITION BY toYYYYMMDD(candle_time);
```

### Materialized View to Populate

```sql
CREATE MATERIALIZED VIEW datastreams.market_ohlcv_1m_mv
TO datastreams.market_ohlcv_1m
AS SELECT
    market,
    toStartOfMinute(block_timestamp) AS candle_time,

    -- STATE functions for storage
    argMinState(spot_price_sol, block_timestamp) AS open_price,
    argMaxState(spot_price_sol, block_timestamp) AS close_price,
    max(spot_price_sol) AS high_price,
    min(spot_price_sol) AS low_price,
    sum(volume_sol) AS volume,
    sumIf(volume_sol, trade_direction = 'BUY') AS buy_volume,
    sumIf(volume_sol, trade_direction = 'SELL') AS sell_volume,
    count() AS trade_count,
    uniqCombinedState(14)(trader) AS unique_traders

FROM datastreams.enriched_trades
GROUP BY market, candle_time;
```

### Query Pattern

```sql
SELECT
    market,
    candle_time,
    -- MERGE functions to get final values
    argMinMerge(open_price) AS open,
    argMaxMerge(close_price) AS close,
    high_price AS high,
    low_price AS low,
    volume,
    trade_count,
    uniqCombinedMerge(14)(unique_traders) AS unique_traders
FROM datastreams.market_ohlcv_1m
WHERE market = {market:String}
  AND candle_time >= {start_time:DateTime64}
  AND candle_time < {end_time:DateTime64}
GROUP BY market, candle_time, high_price, low_price, volume, trade_count
ORDER BY candle_time;
```

---

## 3.2 HyperLogLog for Unique Counts

### Why uniqCombined(14)?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HYPERLOGLOG UNIQUE COUNT ALGORITHM                        │
└─────────────────────────────────────────────────────────────────────────────┘

Problem: Count unique traders across 1M trades
─────────────────────────────────────────────────
Naive approach: Store all trader addresses in a Set
• 1M trades × 44 bytes/address = 44 MB per query
• Distributed: Can't merge sets efficiently

HyperLogLog approach: Store probabilistic sketch
• uniqCombined(14) uses 2^14 = 16,384 registers
• Memory: ~16 KB (fixed, regardless of cardinality)
• Error rate: ~1.04% (acceptable for analytics)
• MERGEABLE: Can combine sketches from different partitions!

uniqCombined(14) breakdown:
├── 14 = precision parameter (2^14 registers)
├── Higher = more accurate but more memory
├── 12 = 4KB, ~1.6% error
├── 14 = 16KB, ~1.0% error (our choice)
├── 16 = 64KB, ~0.4% error
└── 18 = 256KB, ~0.1% error
```

### Trending Analytics Example

```sql
CREATE TABLE datastreams.trending_launchpad_volume_daily
(
    date Date,
    venue LowCardinality(String),

    -- Sum aggregates
    volume_sol AggregateFunction(sum, Decimal128(18)),
    volume_usd AggregateFunction(sum, Decimal128(18)),
    trade_count AggregateFunction(sum, UInt64),

    -- HyperLogLog unique counts
    unique_tokens AggregateFunction(uniqCombined(14), String),
    unique_traders AggregateFunction(uniqCombined(14), String),
    unique_markets AggregateFunction(uniqCombined(14), String)
)
ENGINE = ReplicatedAggregatingMergeTree(...)
ORDER BY (date, venue)
PARTITION BY toYYYYMMDD(date);
```

### Query with Merge

```sql
-- Daily stats by venue
SELECT
    date,
    venue,
    sumMerge(volume_usd) AS total_volume_usd,
    sumMerge(trade_count) AS total_trades,
    uniqCombinedMerge(14)(unique_tokens) AS unique_tokens,
    uniqCombinedMerge(14)(unique_traders) AS unique_traders
FROM datastreams.trending_launchpad_volume_daily
WHERE date >= today() - INTERVAL 7 DAY
GROUP BY date, venue
ORDER BY date DESC, total_volume_usd DESC;

-- Weekly rollup (HyperLogLog merges across days!)
SELECT
    toStartOfWeek(date) AS week,
    venue,
    sumMerge(volume_usd) AS weekly_volume,
    uniqCombinedMerge(14)(unique_traders) AS weekly_unique_traders
FROM datastreams.trending_launchpad_volume_daily
WHERE date >= today() - INTERVAL 30 DAY
GROUP BY week, venue
ORDER BY week DESC;
```

---

## 3.3 SimpleAggregateFunction vs AggregateFunction

### Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│            SimpleAggregateFunction vs AggregateFunction                      │
└─────────────────────────────────────────────────────────────────────────────┘

SimpleAggregateFunction:
├── Stores FINAL value directly
├── Supported functions: any, anyLast, min, max, sum, groupBitAnd/Or/Xor
├── Query: SELECT column_name (no Merge needed)
├── Pros: Simpler queries, smaller storage
└── Cons: Limited function support

AggregateFunction:
├── Stores INTERMEDIATE state
├── Supported functions: ALL aggregate functions
├── Query: SELECT functionMerge(column_name)
├── Pros: Any aggregate, mergeable across partitions
└── Cons: Larger storage, requires Merge in query

WHEN TO USE EACH:
─────────────────────────────────────────────────────────────────────────────
sum, min, max, count → SimpleAggregateFunction (simpler)
avg, median, quantile → AggregateFunction (need intermediate state)
uniqCombined, argMin, argMax → AggregateFunction (complex state)
```

### Example in OHLCV

```sql
-- high_price: Just need max, use SimpleAggregateFunction
high_price SimpleAggregateFunction(max, Decimal128(18)),

-- Query: Just reference it directly
SELECT high_price FROM market_ohlcv_1m;

-- unique_traders: Need HyperLogLog, must use AggregateFunction
unique_traders AggregateFunction(uniqCombined(14), String),

-- Query: Must use Merge function
SELECT uniqCombinedMerge(14)(unique_traders) FROM market_ohlcv_1m;
```

---

# Part 4: Deduplication & Data Quality

## 4.1 cityHash64 Deduplication

### Why We Need Deduplication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DEDUPLICATION SCENARIOS                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Scenario 1: NATS At-Least-Once Delivery
─────────────────────────────────────────
NATS guarantees at-least-once delivery. During reconnection:
• Same message may be delivered 2-3 times
• Without dedup: Duplicate trades in database

Scenario 2: Multi-Hop Trades
─────────────────────────────────────────
One Solana transaction can have multiple hops:
• signature=ABC, hop_index=0 (first swap)
• signature=ABC, hop_index=1 (second swap)
• These are DIFFERENT trades, need different dedup keys

Scenario 3: Replays
─────────────────────────────────────────
During recovery, may replay historical data:
• Same trades inserted again
• Need to detect and skip duplicates
```

### Implementation

```sql
-- MATERIALIZED column: Computed once at insert time
dedup_hash UInt64 MATERIALIZED cityHash64(signature, toString(hop_index))
    CODEC(Delta, ZSTD(1)),

-- Why cityHash64?
-- • Fast: ~3 GB/s hashing speed
-- • 64-bit: Collision probability ≈ 1 in 18 quintillion
-- • Deterministic: Same input always gives same output
```

### Checking for Duplicates

```sql
-- Find potential duplicates (after insert, before merge)
SELECT
    dedup_hash,
    count() AS cnt,
    groupArray(signature)[1] AS example_signature
FROM datastreams.enriched_trades
WHERE block_timestamp >= now() - INTERVAL 1 HOUR
GROUP BY dedup_hash
HAVING cnt > 1
LIMIT 100;

-- If duplicates exist, they'll be removed on next merge
-- Or use FINAL for immediate deduplication:
SELECT * FROM enriched_trades FINAL WHERE trader = 'ABC';
```

---

## 4.2 Dead Letter Queue (DLQ)

### DLQ Purpose

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DLQ DATA FLOW                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Normal flow:
NATS → Queue → MV → enriched_trades ✓

Error flow:
NATS → Queue → MV → ❌ Validation fails → dlq_failed_records

Error types:
• invalid_json: Malformed JSON payload
• invalid_price: price <= 0 or NULL
• invalid_volume: volume <= 0 or NULL
• missing_field: Required field missing
• stale_timestamp: Timestamp > 24h old
• parse_error: Type conversion failed
```

### DLQ Table

```sql
CREATE TABLE datastreams.dlq_failed_records
(
    -- Auto-generated unique ID
    error_id UUID DEFAULT generateUUIDv4() CODEC(ZSTD(3)),

    -- Source identification
    source_stream LowCardinality(String),      -- NATS subject
    source_table LowCardinality(String),       -- Target table

    -- The failed data
    raw_payload String CODEC(ZSTD(3)),         -- Original JSON/RowBinary

    -- Error details
    error_type LowCardinality(String),
    error_message String CODEC(ZSTD(1)),
    error_field Nullable(String) CODEC(ZSTD(1)),

    -- Context
    partition_key Nullable(String) CODEC(ZSTD(1)),
    nats_sequence Nullable(UInt64) CODEC(T64, ZSTD(1)),
    original_timestamp Nullable(DateTime64(9)) CODEC(DoubleDelta, ZSTD(1)),

    -- Tracking
    failed_at DateTime64(3) DEFAULT now64(3) CODEC(DoubleDelta, ZSTD(1)),
    reprocess_attempts UInt8 DEFAULT 0,
    last_reprocess_at Nullable(DateTime64(3)) CODEC(DoubleDelta, ZSTD(1)),
    is_resolved UInt8 DEFAULT 0,

    -- Indexes for monitoring
    INDEX idx_source_stream source_stream TYPE set(50) GRANULARITY 1,
    INDEX idx_source_table source_table TYPE set(50) GRANULARITY 1,
    INDEX idx_error_type error_type TYPE set(10) GRANULARITY 1,
    INDEX idx_resolved is_resolved TYPE set(2) GRANULARITY 1
)
ENGINE = ReplicatedMergeTree(...)
PARTITION BY toYYYYMMDD(failed_at)
ORDER BY (source_stream, source_table, failed_at, error_id)
TTL failed_at + INTERVAL 30 DAY DELETE
SETTINGS ttl_only_drop_parts = 1;
```

### DLQ Monitoring Queries

```sql
-- Error rate by source (last 24h)
SELECT
    source_stream,
    error_type,
    count() AS error_count,
    max(failed_at) AS latest_error,
    min(failed_at) AS earliest_error
FROM datastreams.dlq_failed_records
WHERE failed_at >= now() - INTERVAL 24 HOUR
  AND is_resolved = 0
GROUP BY source_stream, error_type
ORDER BY error_count DESC;

-- Sample failed records for debugging
SELECT
    error_id,
    source_stream,
    error_type,
    error_message,
    substring(raw_payload, 1, 200) AS payload_preview,
    failed_at
FROM datastreams.dlq_failed_records
WHERE error_type = 'invalid_price'
  AND is_resolved = 0
ORDER BY failed_at DESC
LIMIT 10;

-- Mark records as resolved after fixing
ALTER TABLE datastreams.dlq_failed_records
UPDATE is_resolved = 1, last_reprocess_at = now64(3)
WHERE error_type = 'invalid_price'
  AND failed_at >= '2024-01-15'
  AND failed_at < '2024-01-16';
```

---

# Part 5: Index Strategy

## 5.1 Skip Index Types

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SKIP INDEX DECISION TREE                             │
└─────────────────────────────────────────────────────────────────────────────┘

What kind of query filter?
│
├── Exact match on high-cardinality column
│   │
│   └── bloom_filter(0.01)
│       • False positive rate: 1%
│       • Memory: ~10 bytes per granule
│       • Examples: trader, token_mint, signature
│
│       INDEX idx_trader trader TYPE bloom_filter(0.01) GRANULARITY 4
│
├── Exact match on low-cardinality column
│   │
│   └── set(N) where N ≈ unique values
│       • Stores actual values
│       • Memory: N × value_size per granule
│       • Examples: dex (20), stage (5)
│
│       INDEX idx_dex dex TYPE set(20) GRANULARITY 4
│
├── Text search / LIKE queries
│   │
│   └── tokenbf_v1(bloom_size, hash_count, seed)
│       • Token-based bloom filter
│       • For: LIKE '%term%' queries
│       • Examples: token_name search
│
│       INDEX idx_name token_name TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4
│
└── Range queries on non-primary-key column
    │
    └── minmax
        • Stores min/max per granule
        • For: WHERE price > 100

        INDEX idx_price price TYPE minmax GRANULARITY 4
```

## 5.2 Click Trade Index Inventory

| Table | Column | Index Type | Why |
|-------|--------|------------|-----|
| `enriched_trades` | trader | `bloom_filter(0.01)` | High cardinality, exact match |
| `enriched_trades` | token_mint | `bloom_filter(0.01)` | High cardinality, exact match |
| `enriched_trades` | dex | `set(20)` | Low cardinality, exact match |
| `token_metrics` | token_symbol | `set(1000)` | Medium cardinality |
| `token_metrics` | primary_dex | `set(20)` | Low cardinality |
| `discovery` | stage | `set(10)` | Low cardinality |
| `discovery` | token_name | `tokenbf_v1(10240, 3, 0)` | Text search |
| `discovery` | protocols | `bloom_filter(0.01)` | Array contains |

## 5.3 GRANULARITY Setting

```
GRANULARITY = number of granules per index block

Default granule = 8192 rows
GRANULARITY 4 = index covers 4 × 8192 = 32,768 rows per block

Lower GRANULARITY:
├── More precise index
├── More memory usage
├── Better for highly selective queries

Higher GRANULARITY:
├── Less precise index
├── Less memory usage
├── Better for less selective queries

Recommendation:
├── High selectivity (< 1% rows match): GRANULARITY 1-2
├── Medium selectivity (1-10% match): GRANULARITY 4
├── Low selectivity (> 10% match): GRANULARITY 8+
```

---

# Part 6: Query Patterns

## 6.1 Token Metrics Queries

### Current Token State

```sql
-- Get single token (uses ORDER BY index)
SELECT
    token_mint,
    token_symbol,
    price_usd,
    market_cap_usd,
    liquidity_usd,
    holders_count,
    stats_24h_volume_usd,
    stats_24h_price_change_pct,
    ath_price_usd,
    fromUnixTimestamp64Milli(ath_price_timestamp) AS ath_price_time
FROM datastreams.token_metrics FINAL
WHERE token_mint = {token_mint:String};
```

### Top Tokens Leaderboard

```sql
-- Top 100 by 24h volume (full table scan on FINAL)
SELECT
    token_mint,
    token_symbol,
    stats_24h_volume_usd,
    market_cap_usd,
    stats_24h_price_change_pct,
    holders_count
FROM datastreams.token_metrics FINAL
ORDER BY stats_24h_volume_usd DESC
LIMIT 100;
```

### Risk Analysis

```sql
-- Tokens with high risk indicators
SELECT
    token_mint,
    token_symbol,
    bundler_pct,
    snipers_pct,
    insiders_pct,
    rug_risk_pct,
    is_honeypot,
    has_freeze_authority,
    has_mint_authority
FROM datastreams.token_metrics FINAL
WHERE rug_risk_pct > 50
   OR is_honeypot = 1
   OR bundler_pct > 30
ORDER BY rug_risk_pct DESC
LIMIT 100;
```

---

## 6.2 Position & PnL Queries

### Trader Portfolio

```sql
-- All positions for a trader (uses ORDER BY on trader)
SELECT
    trader,
    token_mint,
    token_symbol,
    holdings_token,
    supply_pct,
    cost_basis_usd,
    current_value_usd,
    realized_pnl_usd,
    unrealized_pnl_usd,
    unrealized_pnl_pct,
    total_buys,
    total_sells,
    first_buy_timestamp,
    last_activity_timestamp
FROM datastreams.position_overview FINAL
WHERE trader = {trader:String}
  AND holdings_token > 0  -- Only active positions
ORDER BY current_value_usd DESC;
```

### Portfolio Summary

```sql
-- Aggregate portfolio metrics
SELECT
    trader,
    count() AS total_positions,
    countIf(holdings_token > 0) AS active_positions,
    sum(cost_basis_usd) AS total_cost_basis,
    sum(current_value_usd) AS total_value,
    sum(realized_pnl_usd) AS total_realized_pnl,
    sum(unrealized_pnl_usd) AS total_unrealized_pnl,
    sum(total_fees_paid_sol) AS total_fees,
    -- Portfolio ROI
    round(
        sum(unrealized_pnl_usd) / nullIf(sum(cost_basis_usd), 0) * 100,
        2
    ) AS portfolio_roi_pct,
    -- Win rate
    round(
        countIf(unrealized_pnl_usd > 0) / count() * 100,
        2
    ) AS win_rate_pct
FROM datastreams.position_overview FINAL
WHERE trader = {trader:String}
GROUP BY trader;
```

### Best/Worst Trades

```sql
-- Top 10 winning positions
SELECT
    token_mint,
    token_symbol,
    cost_basis_usd,
    current_value_usd,
    unrealized_pnl_usd,
    unrealized_pnl_pct
FROM datastreams.position_overview FINAL
WHERE trader = {trader:String}
  AND cost_basis_usd > 0
  AND holdings_token > 0
ORDER BY unrealized_pnl_pct DESC
LIMIT 10;

-- Top 10 losing positions
SELECT
    token_mint,
    token_symbol,
    cost_basis_usd,
    current_value_usd,
    unrealized_pnl_usd,
    unrealized_pnl_pct
FROM datastreams.position_overview FINAL
WHERE trader = {trader:String}
  AND cost_basis_usd > 0
  AND holdings_token > 0
ORDER BY unrealized_pnl_pct ASC
LIMIT 10;
```

---

## 6.3 Trade History Queries

### Trades by Token

```sql
-- Recent trades for a token (uses primary ORDER BY)
SELECT
    signature,
    trader,
    trade_direction,
    volume_sol,
    volume_usd,
    execution_price_sol,
    execution_price_usd,
    market_cap_usd,
    block_timestamp
FROM datastreams.enriched_trades
WHERE token_mint = {token_mint:String}
  AND block_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY block_timestamp DESC
LIMIT 100;
```

### Trades by Trader (Uses Projection)

```sql
-- Trader's trade history (uses proj_by_trader projection)
SELECT
    token_mint,
    token_symbol,
    trade_direction,
    volume_sol,
    volume_usd,
    execution_price_usd,
    block_timestamp
FROM datastreams.enriched_trades
WHERE trader = {trader:String}
  AND block_timestamp >= now() - INTERVAL 7 DAY
ORDER BY block_timestamp DESC
LIMIT 100;
```

### Volume Aggregation

```sql
-- 24h volume breakdown for a token
SELECT
    token_mint,
    any(token_symbol) AS token_symbol,

    -- Total volume
    sum(volume_usd) AS total_volume_usd,
    sum(volume_sol) AS total_volume_sol,

    -- Buy/Sell breakdown
    sumIf(volume_usd, trade_direction = 'BUY') AS buy_volume_usd,
    sumIf(volume_usd, trade_direction = 'SELL') AS sell_volume_usd,
    countIf(trade_direction = 'BUY') AS buy_count,
    countIf(trade_direction = 'SELL') AS sell_count,

    -- Unique traders
    uniq(trader) AS unique_traders,

    -- Average trade size
    avg(volume_usd) AS avg_trade_size_usd

FROM datastreams.enriched_trades
WHERE token_mint = {token_mint:String}
  AND block_timestamp >= now() - INTERVAL 24 HOUR
GROUP BY token_mint;
```

---

## 6.4 OHLCV Candlestick Queries

### Get Candles for Charting

```sql
-- 1-minute candles for last hour
SELECT
    market,
    candle_time,
    argMinMerge(open_price) AS open,
    high_price AS high,
    low_price AS low,
    argMaxMerge(close_price) AS close,
    volume,
    trade_count,
    uniqCombinedMerge(14)(unique_traders) AS unique_traders
FROM datastreams.market_ohlcv_1m
WHERE market = {market:String}
  AND candle_time >= now() - INTERVAL 1 HOUR
GROUP BY market, candle_time, high_price, low_price, volume, trade_count
ORDER BY candle_time ASC;
```

### Multi-Timeframe Query

```sql
-- Get appropriate timeframe based on range
-- Range < 6h: 1m candles
-- Range 6h-24h: 5m candles
-- Range 1d-7d: 15m candles
-- Range > 7d: 1h candles

WITH params AS (
    SELECT
        {start_time:DateTime64} AS start_time,
        {end_time:DateTime64} AS end_time,
        dateDiff('hour', start_time, end_time) AS hours_diff
)
SELECT
    candle_time,
    argMinMerge(open_price) AS open,
    high_price AS high,
    low_price AS low,
    argMaxMerge(close_price) AS close,
    volume
FROM (
    SELECT * FROM datastreams.market_ohlcv_1m
    WHERE (SELECT hours_diff FROM params) <= 6

    UNION ALL

    SELECT * FROM datastreams.market_ohlcv_5m
    WHERE (SELECT hours_diff FROM params) > 6
      AND (SELECT hours_diff FROM params) <= 24

    UNION ALL

    SELECT * FROM datastreams.market_ohlcv_15m
    WHERE (SELECT hours_diff FROM params) > 24
      AND (SELECT hours_diff FROM params) <= 168  -- 7 days

    UNION ALL

    SELECT * FROM datastreams.market_ohlcv_1h
    WHERE (SELECT hours_diff FROM params) > 168
)
WHERE market = {market:String}
  AND candle_time >= (SELECT start_time FROM params)
  AND candle_time < (SELECT end_time FROM params)
GROUP BY candle_time, high_price, low_price, volume
ORDER BY candle_time;
```

---

## 6.5 Discovery Queries

### Newly Created Tokens

```sql
SELECT
    token_mint,
    token_symbol,
    token_name,
    stage,
    bonding_curve_pct,
    price_usd,
    market_cap_usd,
    volume_24h_usd,
    holders_count,
    creator,
    created_at
FROM datastreams.discovery FINAL
WHERE stage = 'newly_created'
  AND created_at >= now() - INTERVAL 1 HOUR
ORDER BY created_at DESC
LIMIT 50;
```

### About to Graduate

```sql
SELECT
    token_mint,
    token_symbol,
    bonding_curve_pct,
    market_cap_usd,
    volume_24h_usd,
    holders_count,
    primary_protocol
FROM datastreams.discovery FINAL
WHERE stage = 'about_to_graduate'
  AND bonding_curve_pct >= 80
ORDER BY bonding_curve_pct DESC
LIMIT 50;
```

### Recently Graduated

```sql
SELECT
    token_mint,
    token_symbol,
    market_cap_usd,
    volume_24h_usd,
    graduation_timestamp,
    primary_dex
FROM datastreams.discovery FINAL
WHERE stage = 'graduated'
  AND graduation_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY graduation_timestamp DESC;
```

---

## 6.6 ATH (All-Time High) Queries

### Token ATH Metrics

```sql
SELECT
    token_mint,
    token_symbol,

    -- Current values
    price_usd AS current_price,
    market_cap_usd AS current_mcap,

    -- ATH Price
    ath_price_usd,
    fromUnixTimestamp64Milli(ath_price_timestamp) AS ath_price_time,

    -- ATH Market Cap
    ath_market_cap_usd,
    fromUnixTimestamp64Milli(ath_market_cap_timestamp) AS ath_mcap_time,

    -- Distance from ATH
    round((1 - (price_usd / ath_price_usd)) * 100, 2) AS pct_below_ath_price,
    round((1 - (market_cap_usd / ath_market_cap_usd)) * 100, 2) AS pct_below_ath_mcap

FROM datastreams.token_metrics FINAL
WHERE token_mint = {token_mint:String};
```

### Tokens Near ATH (Potential Breakout)

```sql
SELECT
    token_mint,
    token_symbol,
    market_cap_usd,
    ath_market_cap_usd,
    round((market_cap_usd / ath_market_cap_usd) * 100, 2) AS pct_of_ath
FROM datastreams.token_metrics FINAL
WHERE ath_market_cap_usd > 0
  AND market_cap_usd > 10000  -- Filter tiny tokens
ORDER BY pct_of_ath DESC
LIMIT 50;
```

### New ATH in Last 24h

```sql
SELECT
    token_mint,
    token_symbol,
    ath_market_cap_usd,
    fromUnixTimestamp64Milli(ath_market_cap_timestamp) AS ath_time,
    market_cap_usd
FROM datastreams.token_metrics FINAL
WHERE ath_market_cap_timestamp >= toUnixTimestamp64Milli(now() - INTERVAL 24 HOUR)
ORDER BY ath_market_cap_usd DESC
LIMIT 50;
```

---

# Part 7: Performance Optimization

## 7.1 Query Settings

### Essential Settings

```sql
-- For ReplacingMergeTree FINAL queries (faster dedup)
SET do_not_merge_across_partitions_select_final = 1;

-- Prevent accidental full table scans
SET force_primary_key = 1;

-- Enable query cache for repeated queries
SET use_query_cache = 1;

-- Handle non-deterministic functions in cache
SET query_cache_nondeterministic_function_handling = 'save';

-- Memory limits
SET max_memory_usage = 20000000000;  -- 20 GB

-- Spill to disk for large GROUP BY
SET max_bytes_before_external_group_by = 10000000000;  -- 10 GB

-- Query timeout
SET max_execution_time = 60;  -- 60 seconds
```

### Performance Impact

| Setting | Before | After | Impact |
|---------|--------|-------|--------|
| `do_not_merge_across_partitions_select_final` | 2.5s | 0.3s | 8x faster FINAL |
| `force_primary_key` | (no impact) | Error on full scan | Prevents mistakes |
| `use_query_cache` | 50ms | 5ms (cached) | 10x for repeats |

---

## 7.2 Table Settings

```sql
CREATE TABLE example (...)
ENGINE = ReplicatedMergeTree(...)
...
SETTINGS
    -- Fast TTL cleanup (drop whole parts, not row-by-row)
    ttl_only_drop_parts = 1,

    -- TTL merge interval (24h default, can lower for faster cleanup)
    merge_with_ttl_timeout = 86400,

    -- Index granularity (default 8192, rarely need to change)
    index_granularity = 8192,

    -- Wide parts threshold (10MB default)
    min_bytes_for_wide_part = 10485760,

    -- Storage policy for tiered storage
    storage_policy = 'tiered_storage';
```

---

## 7.3 Expected Query Latencies

| Query Type | Complexity | Expected p50 | Expected p99 |
|------------|------------|--------------|--------------|
| Single token lookup | Simple | 5-10ms | 50ms |
| Trader portfolio (10 positions) | Simple | 10-20ms | 100ms |
| Top 100 tokens by volume | Medium | 50-100ms | 500ms |
| 24h volume aggregation | Medium | 20-50ms | 200ms |
| 1h OHLCV candles (60 points) | Simple | 10-20ms | 100ms |
| Full discovery feed (50 items) | Medium | 30-50ms | 200ms |
| Trending analytics (7 days) | Complex | 100-200ms | 1s |
| Cross-table joins | Complex | 200-500ms | 2s |

---

# Part 8: Anti-Patterns (What NOT to Do)

## 8.1 Schema Anti-Patterns

### ❌ Using Float for Financial Data

```sql
-- ❌ WRONG
CREATE TABLE bad_trades (
    price Float64,      -- Precision loss!
    volume Float64      -- 0.1 + 0.2 != 0.3
);

-- ✅ CORRECT
CREATE TABLE good_trades (
    price Decimal(38, 18),
    volume Decimal(38, 18)
);
```

### ❌ String for Low Cardinality

```sql
-- ❌ WRONG (20 unique DEXes stored as full strings)
dex String,

-- ✅ CORRECT
dex LowCardinality(String),
```

### ❌ Wrong ORDER BY

```sql
-- ❌ WRONG (90% of queries filter by token_mint, but it's not first)
ORDER BY (block_timestamp, token_mint, signature)

-- ✅ CORRECT (most filtered column first)
ORDER BY (token_mint, block_timestamp, signature)
```

### ❌ Missing FINAL on ReplacingMergeTree

```sql
-- ❌ WRONG (returns duplicates before merge)
SELECT * FROM token_metrics WHERE token_mint = 'ABC';

-- ✅ CORRECT
SELECT * FROM token_metrics FINAL WHERE token_mint = 'ABC';
```

---

## 8.2 Query Anti-Patterns

### ❌ SELECT * on Large Tables

```sql
-- ❌ WRONG
SELECT * FROM enriched_trades WHERE trader = 'ABC';

-- ✅ CORRECT (only needed columns)
SELECT signature, volume_usd, block_timestamp
FROM enriched_trades WHERE trader = 'ABC';
```

### ❌ Missing WHERE on Time Column

```sql
-- ❌ WRONG (scans entire table)
SELECT * FROM enriched_trades WHERE trader = 'ABC';

-- ✅ CORRECT (add time filter to limit scan)
SELECT * FROM enriched_trades
WHERE trader = 'ABC'
  AND block_timestamp >= now() - INTERVAL 7 DAY;
```

### ❌ Using FINAL with Large Result Sets

```sql
-- ❌ WRONG (FINAL on millions of rows)
SELECT * FROM token_metrics FINAL ORDER BY volume DESC LIMIT 100;

-- ✅ BETTER (filter first, then FINAL)
SELECT * FROM token_metrics FINAL
WHERE stats_24h_volume_usd > 10000
ORDER BY stats_24h_volume_usd DESC
LIMIT 100;
```

### ❌ Cross-Shard Joins (Future Consideration)

```sql
-- ❌ WRONG (when we have multiple shards)
SELECT * FROM trades t
JOIN positions p ON t.trader = p.trader;
-- This requires shuffling data between shards

-- ✅ CORRECT (join on shard key or use subquery)
SELECT * FROM trades t
WHERE t.trader IN (
    SELECT trader FROM positions WHERE ...
);
```

---

## 8.3 Operational Anti-Patterns

### ❌ Inserting Row-by-Row

```sql
-- ❌ WRONG (1000 individual inserts)
INSERT INTO trades VALUES (...);
INSERT INTO trades VALUES (...);
-- ... 998 more times

-- ✅ CORRECT (batch insert)
INSERT INTO trades VALUES
    (...),
    (...),
    -- ... up to 50,000 rows per batch
    (...);
```

### ❌ Running Expensive Queries Without Limits

```sql
-- ❌ WRONG (no timeout, no row limit)
SELECT * FROM enriched_trades WHERE dex = 'Raydium';

-- ✅ CORRECT
SET max_execution_time = 30;
SET max_rows_to_read = 10000000;
SELECT * FROM enriched_trades WHERE dex = 'Raydium' LIMIT 1000;
```

---

# Part 9: Troubleshooting Runbook

## 9.1 Slow Queries

### Diagnosis

```sql
-- Check query execution stats
SELECT
    query_id,
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage,
    result_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000  -- Queries > 1 second
  AND event_date = today()
ORDER BY query_duration_ms DESC
LIMIT 20;
```

### Common Causes & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `read_rows` very high | Missing primary key filter | Add ORDER BY column to WHERE |
| `memory_usage` high | Large GROUP BY | Add `max_bytes_before_external_group_by` |
| Slow FINAL | Large table, many duplicates | Use `do_not_merge_across_partitions_select_final` |
| Timeout | Too broad query | Add time filter, increase timeout |

### EXPLAIN Analysis

```sql
EXPLAIN indexes = 1
SELECT * FROM enriched_trades
WHERE token_mint = 'ABC'
  AND block_timestamp >= now() - INTERVAL 1 DAY;

-- Look for:
-- • "PrimaryKey" condition (good)
-- • "Skip" indexes being used (good)
-- • Full partition scans (bad)
```

---

## 9.2 Replication Lag

### Diagnosis

```sql
-- Check replication status
SELECT
    database,
    table,
    replica_name,
    is_leader,
    total_replicas,
    active_replicas,
    queue_size,
    inserts_in_queue,
    log_pointer,
    last_queue_update
FROM system.replicas
WHERE database = 'datastreams';
```

### Common Causes & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `queue_size` growing | Slow replica | Check disk I/O, network |
| `active_replicas` < `total_replicas` | Replica down | Restart replica, check ZK |
| High `inserts_in_queue` | Insert backlog | Scale ingestion, batch larger |

---

## 9.3 Disk Space Issues

### Diagnosis

```sql
-- Table sizes
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows,
    count() AS parts
FROM system.parts
WHERE database = 'datastreams' AND active = 1
GROUP BY table
ORDER BY sum(bytes_on_disk) DESC;

-- Partition sizes (for TTL troubleshooting)
SELECT
    table,
    partition,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    min(min_time) AS min_time,
    max(max_time) AS max_time
FROM system.parts
WHERE database = 'datastreams' AND active = 1
GROUP BY table, partition
ORDER BY table, partition;
```

### Common Causes & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Old partitions not dropping | TTL not running | `OPTIMIZE TABLE ... FINAL` |
| Many small parts | Too frequent inserts | Batch inserts, run `OPTIMIZE` |
| Parts on wrong volume | TTL TO VOLUME not running | Check `merge_with_ttl_timeout` |

---

## 9.4 TTL Not Working

### Diagnosis

```sql
-- Check TTL settings
SELECT
    database,
    table,
    engine,
    partition_key,
    sorting_key,
    create_table_query
FROM system.tables
WHERE database = 'datastreams'
  AND table LIKE '%history%';

-- Check parts with expired data
SELECT
    table,
    partition,
    min_time,
    max_time,
    rows,
    formatReadableSize(bytes_on_disk) AS size
FROM system.parts
WHERE database = 'datastreams'
  AND max_time < now() - INTERVAL 30 DAY  -- Should be TTL'd
  AND active = 1
ORDER BY min_time;
```

### Fixes

```sql
-- Force TTL merge
OPTIMIZE TABLE datastreams.dlq_failed_records FINAL;

-- Increase TTL merge frequency (per-table)
ALTER TABLE datastreams.dlq_failed_records
MODIFY SETTING merge_with_ttl_timeout = 3600;  -- 1 hour

-- Check ttl_only_drop_parts is enabled
ALTER TABLE datastreams.dlq_failed_records
MODIFY SETTING ttl_only_drop_parts = 1;
```

---

# Part 10: Capacity Planning

## 10.1 Storage Estimates

### Per-Trade Storage

| Component | Size (Compressed) |
|-----------|-------------------|
| enriched_trades row | ~30 bytes |
| OHLCV (all timeframes) | ~5 bytes |
| Indexes | ~2 bytes |
| **Total per trade** | **~37 bytes** |

### Monthly Projections

| Trades/Day | Trades/Month | Storage/Month | 1 Year |
|------------|--------------|---------------|--------|
| 10,000 | 300,000 | 11 MB | 132 MB |
| 100,000 | 3,000,000 | 111 MB | 1.3 GB |
| 1,000,000 | 30,000,000 | 1.1 GB | 13 GB |
| 10,000,000 | 300,000,000 | 11 GB | 132 GB |

### State Tables (Grow with Cardinality)

| Table | Rows (Est) | Size/Row | Total Size |
|-------|------------|----------|------------|
| token_metrics | 50,000 | 500 bytes | 25 MB |
| position_overview | 500,000 | 200 bytes | 100 MB |
| discovery | 50,000 | 300 bytes | 15 MB |

---

## 10.2 When to Scale

### Add More Replicas When:
- Query latency p99 > 500ms consistently
- CPU utilization > 70% on read replicas
- Read QPS exceeds single-replica capacity

### Add More Shards When:
- Single shard exceeds 10TB
- Write throughput > 100K rows/second sustained
- Query parallelization needed for analytics

### Add Tiered Storage When:
- Hot storage > 500GB
- Historical data > 30 days rarely queried
- Cost optimization needed

---

## 10.3 Cost Analysis

### Storage Costs (AWS Example)

| Storage Type | $/GB/Month | 1TB/Month | 10TB/Month |
|--------------|------------|-----------|------------|
| gp3 SSD | $0.08 | $80 | $800 |
| S3 Standard | $0.023 | $23 | $230 |
| S3 IA | $0.0125 | $12.50 | $125 |

### Compute Costs (Per Node)

| Instance | vCPU | RAM | $/Month (On-Demand) | $/Month (Reserved) |
|----------|------|-----|---------------------|---------------------|
| r6g.large | 2 | 16GB | $73 | $46 |
| r6g.xlarge | 4 | 32GB | $146 | $92 |
| r6g.2xlarge | 8 | 64GB | $292 | $184 |

### 3-Replica Cluster Cost

| Configuration | Compute | Storage (Hot) | Storage (Cold) | Total/Month |
|---------------|---------|---------------|----------------|-------------|
| Small (3x r6g.large, 200GB hot, 1TB cold) | $219 | $16 | $23 | **$258** |
| Medium (3x r6g.xlarge, 500GB hot, 5TB cold) | $438 | $40 | $115 | **$593** |
| Large (3x r6g.2xlarge, 1TB hot, 20TB cold) | $876 | $80 | $460 | **$1,416** |

---

# Part 11: Migration Patterns

## 11.1 Adding New Columns

```sql
-- 1. Add column with default (instant, no rewrite)
ALTER TABLE datastreams.token_metrics
ADD COLUMN new_metric Decimal(38, 18) DEFAULT 0;

-- 2. Update materialized view to populate new column
DROP VIEW IF EXISTS datastreams.token_metrics_mv;
CREATE MATERIALIZED VIEW datastreams.token_metrics_mv
TO datastreams.token_metrics AS
SELECT
    *,
    calculate_new_metric(...) AS new_metric
FROM datastreams.token_metrics_queue;

-- 3. Backfill historical data (optional)
ALTER TABLE datastreams.token_metrics
UPDATE new_metric = calculate_new_metric(...)
WHERE new_metric = 0;
```

## 11.2 Changing ORDER BY

```sql
-- Cannot change ORDER BY directly. Must recreate table.

-- 1. Create new table with new ORDER BY
CREATE TABLE datastreams.enriched_trades_new
(
    -- same schema
)
ENGINE = ReplicatedMergeTree(...)
ORDER BY (new_order_by_columns);

-- 2. Copy data
INSERT INTO datastreams.enriched_trades_new
SELECT * FROM datastreams.enriched_trades;

-- 3. Atomic swap
EXCHANGE TABLES datastreams.enriched_trades AND datastreams.enriched_trades_new;

-- 4. Drop old table
DROP TABLE datastreams.enriched_trades_new;
```

## 11.3 Rebuilding Aggregates

```sql
-- 1. Drop materialized view (stops new inserts to aggregate)
DROP VIEW IF EXISTS datastreams.market_ohlcv_1m_mv;

-- 2. Create shadow table
CREATE TABLE datastreams.market_ohlcv_1m_new AS datastreams.market_ohlcv_1m;

-- 3. Rebuild from source data
INSERT INTO datastreams.market_ohlcv_1m_new
SELECT
    market,
    toStartOfMinute(block_timestamp) AS candle_time,
    argMinState(spot_price_sol, block_timestamp) AS open_price,
    argMaxState(spot_price_sol, block_timestamp) AS close_price,
    max(spot_price_sol) AS high_price,
    min(spot_price_sol) AS low_price,
    sum(volume_sol) AS volume,
    count() AS trade_count,
    uniqCombinedState(14)(trader) AS unique_traders
FROM datastreams.enriched_trades
WHERE block_timestamp >= now() - INTERVAL 7 DAY  -- Rebuild window
GROUP BY market, candle_time;

-- 4. Atomic swap
EXCHANGE TABLES datastreams.market_ohlcv_1m_new AND datastreams.market_ohlcv_1m;

-- 5. Cleanup
DROP TABLE datastreams.market_ohlcv_1m_new;

-- 6. Recreate MV
CREATE MATERIALIZED VIEW datastreams.market_ohlcv_1m_mv
TO datastreams.market_ohlcv_1m AS
SELECT ... FROM datastreams.enriched_trades_queue;
```

---

# Part 12: Security

## 12.1 User Roles

```sql
-- Read-only API user (for backend queries)
CREATE USER api_readonly
IDENTIFIED WITH sha256_password BY 'secure_password'
DEFAULT ROLE readonly_role
SETTINGS
    readonly = 1,
    max_execution_time = 30,
    max_rows_to_read = 10000000,
    max_result_rows = 10000;

-- Admin user (for operations)
CREATE USER admin_user
IDENTIFIED WITH sha256_password BY 'secure_admin_password'
DEFAULT ROLE admin_role;

-- Roles
CREATE ROLE readonly_role;
GRANT SELECT ON datastreams.* TO readonly_role;

CREATE ROLE admin_role;
GRANT ALL ON datastreams.* TO admin_role;
```

## 12.2 Quotas

```sql
-- Rate limiting for API users
CREATE QUOTA api_quota
KEYED BY user_name
FOR INTERVAL 1 MINUTE MAX
    queries = 100,
    result_rows = 1000000
FOR INTERVAL 1 HOUR MAX
    queries = 1000,
    result_rows = 10000000
TO api_readonly;
```

## 12.3 Row-Level Security (Future)

```sql
-- Example: Users can only see their own positions
CREATE ROW POLICY user_positions_policy
ON datastreams.position_overview
FOR SELECT
USING trader = currentUser()
TO api_users;
```

---

# Part 13: Backup & Recovery

## 13.1 Backup Strategy

### What to Backup

| Priority | Data | Method | Frequency |
|----------|------|--------|-----------|
| Critical | State tables (token_metrics, positions) | Snapshot | Daily |
| High | Trade history (enriched_trades) | Incremental | Hourly |
| Medium | Aggregates (OHLCV) | Can rebuild from source | Weekly |
| Low | DLQ | Can replay from NATS | N/A |

### Backup Commands

```bash
# Full backup to S3
clickhouse-backup create --tables="datastreams.*" --backup-name="daily_$(date +%Y%m%d)"
clickhouse-backup upload "daily_$(date +%Y%m%d)"

# Incremental backup (only new parts)
clickhouse-backup create-increment --tables="datastreams.enriched_trades" \
    --backup-name="trades_$(date +%Y%m%d_%H)"
```

## 13.2 Recovery

```bash
# Download backup
clickhouse-backup download "daily_20240115"

# Restore specific table
clickhouse-backup restore --tables="datastreams.token_metrics" "daily_20240115"

# Restore all tables
clickhouse-backup restore "daily_20240115"
```

---

# Part 14: Testing

## 14.1 Query Testing

### EXPLAIN Before Production

```sql
-- Always EXPLAIN expensive queries
EXPLAIN indexes = 1, actions = 1
SELECT ...;

-- Check for:
-- 1. Primary key usage (should see "PrimaryKey" condition)
-- 2. Skip index usage (should see "Index" condition)
-- 3. Part pruning (should see "Selected N parts")
```

### Benchmarking

```sql
-- Run query multiple times, measure p50/p99
SELECT
    quantile(0.5)(query_duration_ms) AS p50_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms,
    avg(read_rows) AS avg_rows_read,
    avg(memory_usage) AS avg_memory
FROM system.query_log
WHERE query_id LIKE 'benchmark_%'
  AND type = 'QueryFinish'
GROUP BY query;
```

## 14.2 Load Testing

```bash
# Using clickhouse-benchmark
clickhouse-benchmark --query="SELECT * FROM token_metrics FINAL LIMIT 100" \
    --concurrency=10 --iterations=1000
```

---

# Appendix A: Quick Reference

## A.1 Common Query Templates

```sql
-- Token lookup
SELECT * FROM token_metrics FINAL WHERE token_mint = {mint:String};

-- Trader portfolio
SELECT * FROM position_overview FINAL WHERE trader = {trader:String};

-- Recent trades
SELECT * FROM enriched_trades
WHERE token_mint = {mint:String}
  AND block_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY block_timestamp DESC LIMIT 100;

-- OHLCV candles
SELECT candle_time, argMinMerge(open_price) AS open, argMaxMerge(close_price) AS close
FROM market_ohlcv_1m
WHERE market = {market:String}
GROUP BY candle_time ORDER BY candle_time;

-- Top tokens by volume
SELECT * FROM token_metrics FINAL
ORDER BY stats_24h_volume_usd DESC LIMIT 100;
```

## A.2 Codec Cheat Sheet

| Column Type | Codec |
|-------------|-------|
| Timestamps | `DoubleDelta, ZSTD(1)` |
| Monotonic integers (slot, sequence) | `Delta, ZSTD(1)` |
| Counters | `T64, ZSTD(1)` |
| Addresses (44 chars) | `ZSTD(3)` |
| Decimals | `ZSTD(1)` |

## A.3 Engine Cheat Sheet

| Use Case | Engine |
|----------|--------|
| Current state (latest per key) | ReplacingMergeTree |
| Append-only events | MergeTree |
| Simple counters | SummingMergeTree |
| Complex aggregates (OHLC, unique counts) | AggregatingMergeTree |

## A.4 Index Cheat Sheet

| Query Pattern | Index Type |
|---------------|------------|
| Exact match, high cardinality | `bloom_filter(0.01)` |
| Exact match, low cardinality | `set(N)` |
| Text search (LIKE '%x%') | `tokenbf_v1(...)` |
| Range queries | `minmax` |

---

## References

- [CryptoHouse](https://github.com/ClickHouse/CryptoHouse) - Solana blockchain analytics (70+ patterns)
- [StockHouse](https://github.com/ClickHouse/stockhouse) - Real-time market data
- [ClickHouse Documentation](https://clickhouse.com/docs)
- [BLOG_10_CLICKHOUSE_OPTIMIZATION.md](./BLOG_10_CLICKHOUSE_OPTIMIZATION.md) - Optimization patterns summary
