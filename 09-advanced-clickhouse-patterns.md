# Advanced ClickHouse Patterns for Click

## Overview

This document covers advanced ClickHouse patterns and best practices:

**Core Patterns (Sections 1-8)**
1. OHLCV candle generation for multiple timeframes
2. Chained/cascading materialized views
3. Lazy materialization optimization
4. PREWHERE deep dive
5. Advanced indexing strategies
6. Chained MV best practices
7. Complete Click implementation
8. AggregatingMergeTree stale data handling

**Optimization Best Practices (Sections 9-19)**
9. JOIN optimization
10. Dictionary patterns
11. Async inserts
12. Memory management
13. Query cache
14. Partitioning strategies
15. Distributed queries & sharding
16. Deduplication strategies
17. ClickHouse 2025 features
18. Compression codecs
19. Nullable antipattern

---

## 1. OHLCV Candle Generation

### 1.1 Understanding OHLCV Aggregation

OHLCV (Open, High, Low, Close, Volume) candles require special aggregation:
- **Open**: First price in the period
- **High**: Maximum price in the period
- **Low**: Minimum price in the period
- **Close**: Last price in the period
- **Volume**: Sum of all volume in the period

### 1.2 Using argMin/argMax for Candles

ClickHouse provides `argMin` and `argMax` functions perfect for OHLCV:

```sql
-- argMin(value, timestamp) returns value where timestamp is minimum (first)
-- argMax(value, timestamp) returns value where timestamp is maximum (last)

SELECT
    toStartOfMinute(tx_timestamp) AS candle_timestamp,
    token_address,
    argMin(price_usd, tx_timestamp) AS open,   -- First price
    max(price_usd) AS high,                     -- Highest price
    min(price_usd) AS low,                      -- Lowest price
    argMax(price_usd, tx_timestamp) AS close,  -- Last price
    sum(trade_total_usd) AS volume,
    count() AS trade_count
FROM trades_nats_v3
WHERE token_address = 'ABC123'
  AND tx_timestamp >= now() - INTERVAL 1 DAY
GROUP BY token_address, candle_timestamp
ORDER BY candle_timestamp;
```

### 1.3 Target Table Schema for OHLCV

```sql
-- Use AggregatingMergeTree for continuous aggregation
CREATE TABLE ohlcv_1m
(
    token_address String,
    candle_timestamp DateTime64(3),

    -- Store aggregate states for proper merging
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, candle_timestamp);
```

### 1.4 Materialized View for 1-Minute Candles

```sql
CREATE MATERIALIZED VIEW mv_ohlcv_1m TO ohlcv_1m
AS SELECT
    token_address,
    toStartOfMinute(tx_timestamp) AS candle_timestamp,

    -- Use State suffix for AggregateFunction columns
    argMinState(price_usd, tx_timestamp) AS open,
    max(price_usd) AS high,
    min(price_usd) AS low,
    argMaxState(price_usd, tx_timestamp) AS close,
    sum(trade_total_usd) AS volume,
    count() AS trade_count
FROM trades_nats_v3
GROUP BY token_address, candle_timestamp;
```

### 1.5 Querying Aggregated OHLCV Data

```sql
-- Use Merge suffix to get final values
SELECT
    token_address,
    candle_timestamp,
    argMinMerge(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_1m
WHERE token_address = 'ABC123'
GROUP BY token_address, candle_timestamp
ORDER BY candle_timestamp;
```

---

## 2. Multi-Timeframe OHLCV with Chained MVs

### 2.1 The MV-on-MV Pattern

**Key principle**: Only the finest-grained MV (1-second or 1-minute) reads from raw trades. All larger timeframes build on smaller timeframes.

```
trades_nats_v3 (raw)
       │
       ▼
  mv_ohlcv_1s (1-second candles)
       │
       ├──────────────────────────────┐
       ▼                              ▼
  mv_ohlcv_1m (1-minute)         mv_ohlcv_5s (5-second)
       │
       ├──────────────────────────────┐
       ▼                              ▼
  mv_ohlcv_5m (5-minute)         mv_ohlcv_15m (15-minute)
       │
       ▼
  mv_ohlcv_1h (1-hour)
       │
       ▼
  mv_ohlcv_1d (1-day)
```

### 2.2 Complete Implementation

#### Step 1: Create Target Tables

```sql
-- 1-minute candles (from raw trades)
CREATE TABLE ohlcv_1m
(
    token_address String,
    candle_timestamp DateTime64(3),
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, candle_timestamp);

-- 5-minute candles (from 1-minute)
CREATE TABLE ohlcv_5m
(
    token_address String,
    candle_timestamp DateTime64(3),
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, candle_timestamp);

-- 1-hour candles (from 5-minute)
CREATE TABLE ohlcv_1h
(
    token_address String,
    candle_timestamp DateTime64(3),
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, candle_timestamp);

-- 1-day candles (from 1-hour)
CREATE TABLE ohlcv_1d
(
    token_address String,
    candle_timestamp Date,
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYear(candle_timestamp)
ORDER BY (token_address, candle_timestamp);
```

#### Step 2: Create Chained Materialized Views

```sql
-- Level 1: Raw trades → 1-minute candles
CREATE MATERIALIZED VIEW mv_ohlcv_1m TO ohlcv_1m
AS SELECT
    token_address,
    toStartOfMinute(tx_timestamp) AS candle_timestamp,
    argMinState(price_usd, tx_timestamp) AS open,
    max(price_usd) AS high,
    min(price_usd) AS low,
    argMaxState(price_usd, tx_timestamp) AS close,
    sum(trade_total_usd) AS volume,
    toUInt64(count()) AS trade_count
FROM trades_nats_v3
GROUP BY token_address, candle_timestamp;

-- Level 2: 1-minute → 5-minute candles
CREATE MATERIALIZED VIEW mv_ohlcv_5m TO ohlcv_5m
AS SELECT
    token_address,
    toStartOfFiveMinutes(candle_timestamp) AS candle_timestamp,
    argMinMergeState(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMergeState(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_1m
GROUP BY token_address, candle_timestamp;

-- Level 3: 5-minute → 1-hour candles
CREATE MATERIALIZED VIEW mv_ohlcv_1h TO ohlcv_1h
AS SELECT
    token_address,
    toStartOfHour(candle_timestamp) AS candle_timestamp,
    argMinMergeState(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMergeState(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_5m
GROUP BY token_address, candle_timestamp;

-- Level 4: 1-hour → 1-day candles
CREATE MATERIALIZED VIEW mv_ohlcv_1d TO ohlcv_1d
AS SELECT
    token_address,
    toDate(candle_timestamp) AS candle_timestamp,
    argMinMergeState(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMergeState(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_1h
GROUP BY token_address, candle_timestamp;
```

### 2.3 All 14 Timeframes for Click

Click supports 14 timeframes: 1s, 5s, 15s, 30s, 1m, 5m, 15m, 30m, 1h, 4h, 6h, 12h, 1d, 1w

**Recommended hierarchy:**

```
trades_nats_v3
       │
       ▼
    ohlcv_1s ──────────────────────────────────────────────────┐
       │                                                        │
       ├─► ohlcv_5s                                            │
       │      │                                                 │
       │      └─► ohlcv_15s                                    │
       │             │                                          │
       │             └─► ohlcv_30s                             │
       │                                                        │
       └─► ohlcv_1m ◄───────────────────────────────────────────┘
              │
              ├─► ohlcv_5m
              │      │
              │      └─► ohlcv_15m
              │             │
              │             └─► ohlcv_30m
              │
              └─► ohlcv_1h
                     │
                     ├─► ohlcv_4h
                     │
                     ├─► ohlcv_6h
                     │      │
                     │      └─► ohlcv_12h
                     │
                     └─► ohlcv_1d
                            │
                            └─► ohlcv_1w
```

### 2.4 Alternative: Single Table with All Timeframes

For simpler queries, store all timeframes in one table:

```sql
CREATE TABLE ohlcv_all_timeframes
(
    token_address String,
    timeframe LowCardinality(String),  -- '1s', '1m', '5m', etc.
    candle_timestamp DateTime64(3),
    open Decimal64(18),
    high Decimal64(18),
    low Decimal64(18),
    close Decimal64(18),
    volume Decimal64(6),
    trade_count UInt64
)
ENGINE = ReplacingMergeTree()
PARTITION BY (timeframe, toYYYYMM(candle_timestamp))
ORDER BY (token_address, timeframe, candle_timestamp);
```

### 2.5 Performance Considerations for Chained MVs

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Insert latency | Linear increase per chain level | Keep chain depth ≤ 4 levels |
| Storage overhead | Each MV writes to separate table | Use aggressive TTL on fine-grained tables |
| Complexity | More tables to manage | Use consistent naming conventions |
| Failure propagation | Error in one MV affects downstream | Monitor all MVs independently |

**Best practice**: Limit materialized views to ~10 per source table.

---

## 3. Lazy Materialization

### 3.1 What is Lazy Materialization?

Introduced in ClickHouse 25.4, lazy materialization defers reading large columns until they're actually needed in the query execution plan.

**Before lazy materialization:**
```
1. Read ALL columns needed for SELECT
2. Apply WHERE filter
3. Sort (ORDER BY)
4. Apply LIMIT
```

**With lazy materialization:**
```
1. Read only filter columns (WHERE)
2. Apply filter
3. Read only sort columns (ORDER BY)
4. Sort and apply LIMIT
5. Read remaining SELECT columns ONLY for final rows
```

### 3.2 Performance Impact

Example query:
```sql
SELECT id, big_string_col1, big_string_col2
FROM big_table
ORDER BY rand()
LIMIT 5;
```

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Query time | 219 seconds | 139 ms | **1,576x faster** |
| Data read | 40GB | 1GB | **40x less** |
| Memory usage | 30GB | 100MB | **300x less** |

### 3.3 Enabling Lazy Materialization

```sql
-- Enable for session
SET query_plan_optimize_lazy_materialization = 1;

-- Enable globally in config
-- <query_plan_optimize_lazy_materialization>1</query_plan_optimize_lazy_materialization>
```

### 3.4 When Lazy Materialization Helps

| Query Pattern | Benefit |
|---------------|---------|
| `SELECT many_columns ... LIMIT N` | Large columns only read for N rows |
| `ORDER BY col LIMIT N` | Only reads sort column fully |
| Top-N queries | Dramatic I/O reduction |
| Queries with large string columns | Defers string reads |

### 3.5 How It Works with Other Optimizations

```
Layer 1: Primary Index
    └─► Prunes granules by primary key

Layer 2: Skip Indexes
    └─► Further prunes by secondary indexes

Layer 3: PREWHERE
    └─► Filters rows before reading all columns

Layer 4: Lazy Materialization
    └─► Defers reading large columns until needed

Layer 5: LIMIT
    └─► Stops processing after N rows
```

Each layer cuts I/O further. Together they minimize data read.

### 3.6 Click-Specific Application

For the trades table with wallet addresses:
```sql
-- This query benefits greatly from lazy materialization
SELECT
    tx_signature,           -- Small, read early
    token_address,          -- Small, read early for filter
    trader_wallet,          -- Medium, deferred
    trade_total_usd,        -- Small, read early for sort
    tx_timestamp
FROM trades_nats_v3
WHERE token_address = 'ABC123'
ORDER BY trade_total_usd DESC
LIMIT 100;

-- Execution with lazy materialization:
-- 1. Read token_address (for WHERE)
-- 2. Filter rows
-- 3. Read trade_total_usd (for ORDER BY)
-- 4. Sort and take top 100
-- 5. Read tx_signature, trader_wallet, tx_timestamp ONLY for 100 rows
```

---

## 4. PREWHERE Deep Dive

### 4.1 PREWHERE vs WHERE

| Aspect | WHERE | PREWHERE |
|--------|-------|----------|
| Column reading | All columns read first | Filter columns read first |
| I/O | Higher | Lower |
| Memory | Higher | Lower |
| When applied | After column read | Before full column read |

### 4.2 Automatic PREWHERE (Default)

ClickHouse automatically moves WHERE conditions to PREWHERE when beneficial:

```sql
-- This query:
SELECT * FROM trades WHERE token_address = 'ABC123';

-- Is automatically rewritten to:
SELECT * FROM trades PREWHERE token_address = 'ABC123';
```

Controlled by: `optimize_move_to_prewhere = 1` (default ON)

### 4.3 Multi-Step PREWHERE (v23.2+)

Process multiple filter columns in optimal order:

```sql
-- Enable multi-step PREWHERE
SET enable_multiple_prewhere_read_steps = 1;
SET move_all_conditions_to_prewhere = 1;

-- Query with multiple filters:
SELECT * FROM trades
WHERE token_address = 'ABC123'
  AND trade_type = 'buy'
  AND trade_total_usd > 1000;

-- ClickHouse processes in order of column size:
-- 1. trade_type (smallest - Enum8)
-- 2. token_address (medium - String)
-- 3. trade_total_usd (for remaining rows)
```

### 4.4 Verifying PREWHERE Usage

```sql
-- Check if PREWHERE is being applied
EXPLAIN SYNTAX
SELECT * FROM trades_nats_v3
WHERE token_address = 'ABC123';

-- Output shows PREWHERE if optimized:
-- SELECT ... FROM trades_nats_v3 PREWHERE token_address = 'ABC123'
```

### 4.5 Manual PREWHERE

For complex queries, you can manually specify PREWHERE:

```sql
-- Manual PREWHERE when automatic doesn't choose optimally
SELECT *
FROM trades_nats_v3
PREWHERE token_address = 'ABC123'  -- Filter first
WHERE tx_timestamp > now() - INTERVAL 1 HOUR;  -- Then time filter
```

---

## 5. Advanced Indexing Strategies

### 5.1 Index Types Summary

| Index Type | Best For | Storage Overhead | Insert Impact |
|------------|----------|------------------|---------------|
| **minmax** | Range queries, sorted data | Very low | Minimal |
| **bloom_filter** | Equality checks, high cardinality | Medium | Medium |
| **set(N)** | Low cardinality (< N values) | Low | Low |
| **ngrambf_v1** | LIKE '%pattern%' | High | High |
| **tokenbf_v1** | Full-text word search | High | High |

### 5.2 Index Selection Decision Tree

```
Is the column in ORDER BY?
├── YES → Don't add skip index (primary key handles it)
└── NO → Continue...

What type of queries?
├── Equality (= 'value')
│   └── High cardinality?
│       ├── YES → bloom_filter
│       └── NO (< 1000 values) → set(1000)
├── Range (>, <, BETWEEN)
│   └── Is data naturally ordered?
│       ├── YES → minmax
│       └── NO → Consider projection instead
├── LIKE '%pattern%'
│   └── ngrambf_v1(3, 256, 2, 0)  -- 3-gram, 256 bloom size
└── Word search (full text)
    └── tokenbf_v1(10240, 3, 0)
```

### 5.3 Click-Specific Index Recommendations

```sql
-- trades_nats_v3 table indexes

-- 1. Wallet lookups (high cardinality, equality)
ALTER TABLE trades_nats_v3
    ADD INDEX idx_wallet bloom_filter(0.01) (trader_wallet)
    GRANULARITY 4;

-- 2. Price range queries (sorted data)
ALTER TABLE trades_nats_v3
    ADD INDEX idx_price minmax (price_usd)
    GRANULARITY 1;

-- 3. Trade type filter (low cardinality)
ALTER TABLE trades_nats_v3
    ADD INDEX idx_trade_type set(10) (trade_type)
    GRANULARITY 8;

-- 4. DEX source filter (low cardinality)
ALTER TABLE trades_nats_v3
    ADD INDEX idx_dex set(50) (dex_source)
    GRANULARITY 4;

-- Materialize indexes for existing data
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_wallet;
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_price;
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_trade_type;
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_dex;
```

### 5.4 Granularity Tuning

| Granularity | Precision | Storage | Best For |
|-------------|-----------|---------|----------|
| 1 | Highest | Highest | Highly selective filters |
| 4 | High | Medium | Default choice |
| 8 | Medium | Lower | Low selectivity filters |
| 16+ | Low | Lowest | Very large tables |

**Rule of thumb**: Start with GRANULARITY 4, then tune based on:
- If index isn't skipping enough → decrease granularity
- If insert performance degrades → increase granularity

### 5.5 Verifying Index Effectiveness

```sql
-- Method 1: EXPLAIN with indexes
EXPLAIN indexes = 1
SELECT count() FROM trades_nats_v3
WHERE trader_wallet = 'ABC123';

-- Look for: "Skip: X/Y granules by minmax/bloom_filter"

-- Method 2: Trace logs
SET send_logs_level = 'trace';
SELECT count() FROM trades_nats_v3
WHERE trader_wallet = 'ABC123';
-- Check logs for "Index ... dropped X/Y granules"

-- Method 3: Query metrics
SELECT
    query,
    ProfileEvents['SelectedMarks'] AS total_marks,
    ProfileEvents['SkippedMarks'] AS skipped_marks,
    ProfileEvents['SelectedRows'] AS rows_read,
    result_rows
FROM system.query_log
WHERE query LIKE '%trader_wallet%'
ORDER BY event_time DESC
LIMIT 5;
```

### 5.6 Projections as Index Alternative

For queries that don't fit well with skip indexes, use projections:

```sql
-- Projection for wallet-based queries
ALTER TABLE trades_nats_v3 ADD PROJECTION proj_by_wallet (
    SELECT *
    ORDER BY (trader_wallet, tx_timestamp)
);
ALTER TABLE trades_nats_v3 MATERIALIZE PROJECTION proj_by_wallet;

-- Projection for DEX analysis
ALTER TABLE trades_nats_v3 ADD PROJECTION proj_by_dex (
    SELECT
        dex_source,
        token_address,
        tx_timestamp,
        trade_total_usd,
        trade_type
    ORDER BY (dex_source, token_address, tx_timestamp)
);
ALTER TABLE trades_nats_v3 MATERIALIZE PROJECTION proj_by_dex;
```

### 5.7 Index vs Projection Decision

| Scenario | Use Index | Use Projection |
|----------|-----------|----------------|
| Single column filter | ✅ | |
| Multiple column filter | | ✅ (different ORDER BY) |
| High selectivity (< 1%) | ✅ | |
| Low selectivity (> 10%) | | ✅ |
| Storage constrained | ✅ | |
| Query latency critical | | ✅ |

---

## 6. Chained Materialized Views Best Practices

### 6.1 Architecture Patterns

#### Pattern 1: Linear Chain (ETL Pipeline)
```
Raw Data → Clean/Filter → Enrich → Aggregate
```

```sql
-- Step 1: Raw ingestion (using Null table to discard raw)
CREATE TABLE raw_events (
    data String
) ENGINE = Null;

-- Step 2: Parse and clean
CREATE TABLE parsed_events (...) ENGINE = MergeTree() ...;

CREATE MATERIALIZED VIEW mv_parse TO parsed_events AS
SELECT
    JSONExtract(data, 'token', 'String') AS token_address,
    JSONExtract(data, 'price', 'Float64') AS price
FROM raw_events
WHERE JSONHas(data, 'token');  -- Filter malformed

-- Step 3: Enrich (join with dictionary)
CREATE TABLE enriched_events (...) ENGINE = MergeTree() ...;

CREATE MATERIALIZED VIEW mv_enrich TO enriched_events AS
SELECT
    pe.*,
    dictGet('token_metadata', 'name', pe.token_address) AS token_name
FROM parsed_events AS pe;

-- Step 4: Aggregate
CREATE TABLE hourly_metrics (...) ENGINE = SummingMergeTree() ...;

CREATE MATERIALIZED VIEW mv_aggregate TO hourly_metrics AS
SELECT
    toStartOfHour(timestamp) AS hour,
    token_address,
    sum(volume) AS volume
FROM enriched_events
GROUP BY hour, token_address;
```

#### Pattern 2: Fan-Out (One Source, Multiple Destinations)
```
Raw Trades ─┬─► Volume Aggregates
            ├─► OHLCV Candles
            ├─► Wallet Activity
            └─► Token Metrics
```

```sql
-- Single source table
-- Multiple MVs reading from same source

CREATE MATERIALIZED VIEW mv_volume TO volume_hourly AS
SELECT ... FROM trades_nats_v3 GROUP BY ...;

CREATE MATERIALIZED VIEW mv_ohlcv TO ohlcv_1m AS
SELECT ... FROM trades_nats_v3 GROUP BY ...;

CREATE MATERIALIZED VIEW mv_wallet TO wallet_activity AS
SELECT ... FROM trades_nats_v3 GROUP BY ...;
```

#### Pattern 3: Hierarchical Rollup (Time-Series)
```
1-second → 1-minute → 5-minute → 1-hour → 1-day
```
(See OHLCV section above)

### 6.2 Important Limitations

#### Limitation 1: No Merged Data in Chains
```sql
-- WARNING: CollapsingMergeTree, ReplacingMergeTree, SummingMergeTree
-- The downstream MV receives the INSERT batch, NOT the merged result!

-- Example problem:
CREATE TABLE source (
    id UInt64,
    value UInt64
) ENGINE = SummingMergeTree() ORDER BY id;

CREATE MATERIALIZED VIEW mv_downstream TO target AS
SELECT id, value FROM source;

-- Insert: (1, 100), (1, 50)
-- source after merge: (1, 150)
-- BUT mv_downstream receives: (1, 100), (1, 50) separately!
```

**Solution**: Use `-Merge` suffix functions in downstream MVs:
```sql
CREATE MATERIALIZED VIEW mv_downstream TO target AS
SELECT
    id,
    sumMerge(value) AS value  -- Proper aggregation
FROM source
GROUP BY id;
```

#### Limitation 2: Insert Performance
Each chained MV adds latency. Measure impact:
```sql
-- Check MV overhead
SELECT
    view_name,
    avg(view_duration_ms) AS avg_duration,
    max(view_duration_ms) AS max_duration
FROM system.query_views_log
GROUP BY view_name
ORDER BY avg_duration DESC;
```

#### Limitation 3: Error Propagation
If one MV fails, downstream MVs don't receive data.

**Mitigation**:
- Monitor `system.query_views_log` for failures
- Use separate chains for critical vs non-critical data

### 6.3 Using Null Tables for Intermediate Steps

When you don't need to query intermediate data:

```sql
-- Null table discards data after MVs process it
CREATE TABLE intermediate_null (
    token_address String,
    price Decimal64(18),
    volume Decimal64(6)
) ENGINE = Null;

-- MV reads from Null table
CREATE MATERIALIZED VIEW mv_aggregate TO final_aggregates AS
SELECT ...
FROM intermediate_null
GROUP BY ...;

-- Data flow: Source → Null Table → MV → Final Table
-- Null table takes no storage
```

### 6.4 Monitoring Chained MVs

```sql
-- Check MV health
SELECT
    database,
    table AS mv_name,
    exception,
    exception_code,
    event_time
FROM system.query_views_log
WHERE type = 'ExceptionAfterStart'
  AND event_time > now() - INTERVAL 1 HOUR
ORDER BY event_time DESC;

-- Check MV latency
SELECT
    view_name,
    count() AS executions,
    avg(view_duration_ms) AS avg_ms,
    max(view_duration_ms) AS max_ms,
    sum(written_rows) AS total_rows
FROM system.query_views_log
WHERE event_time > now() - INTERVAL 1 HOUR
GROUP BY view_name;
```

---

## 7. Complete Click Implementation

### 7.1 OHLCV Architecture for Click

```sql
-- =============================================
-- CLICK OHLCV IMPLEMENTATION
-- =============================================

-- Base table: 1-minute candles from trades
CREATE TABLE ohlcv_1m
(
    token_address String,
    candle_timestamp DateTime64(3),
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume_usd SimpleAggregateFunction(sum, Decimal64(6)),
    volume_token SimpleAggregateFunction(sum, Decimal128(18)),
    trade_count SimpleAggregateFunction(sum, UInt64),
    buy_volume SimpleAggregateFunction(sum, Decimal64(6)),
    sell_volume SimpleAggregateFunction(sum, Decimal64(6))
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, candle_timestamp)
TTL candle_timestamp + INTERVAL 90 DAY;

-- MV: trades → 1m candles
CREATE MATERIALIZED VIEW mv_ohlcv_1m TO ohlcv_1m AS
SELECT
    token_address,
    toStartOfMinute(tx_timestamp) AS candle_timestamp,
    argMinState(price_usd, tx_timestamp) AS open,
    max(price_usd) AS high,
    min(price_usd) AS low,
    argMaxState(price_usd, tx_timestamp) AS close,
    sum(trade_total_usd) AS volume_usd,
    sum(token_amount) AS volume_token,
    toUInt64(count()) AS trade_count,
    sumIf(trade_total_usd, trade_type = 'buy') AS buy_volume,
    sumIf(trade_total_usd, trade_type = 'sell') AS sell_volume
FROM trades_nats_v3
GROUP BY token_address, candle_timestamp;

-- Similar tables for 5m, 15m, 30m, 1h, 4h, 6h, 12h, 1d, 1w
-- Each reads from the previous level
```

### 7.2 Unified Query View

```sql
-- Create view for easy querying across timeframes
CREATE VIEW v_ohlcv AS
SELECT
    '1m' AS timeframe,
    token_address,
    candle_timestamp,
    argMinMerge(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume_usd) AS volume
FROM ohlcv_1m
GROUP BY token_address, candle_timestamp

UNION ALL

SELECT
    '5m' AS timeframe,
    token_address,
    candle_timestamp,
    argMinMerge(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume_usd) AS volume
FROM ohlcv_5m
GROUP BY token_address, candle_timestamp

-- ... repeat for other timeframes
;

-- Query any timeframe
SELECT * FROM v_ohlcv
WHERE token_address = 'ABC123'
  AND timeframe = '1h'
  AND candle_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY candle_timestamp;
```

### 7.3 Index Strategy for Click Tables

```sql
-- trades_nats_v3 indexes
ALTER TABLE trades_nats_v3
    ADD INDEX idx_wallet bloom_filter(0.01) (trader_wallet) GRANULARITY 4,
    ADD INDEX idx_price minmax (price_usd) GRANULARITY 1,
    ADD INDEX idx_trade_type set(10) (trade_type) GRANULARITY 8;

-- Projection for wallet queries
ALTER TABLE trades_nats_v3 ADD PROJECTION proj_by_wallet (
    SELECT * ORDER BY (trader_wallet, tx_timestamp)
);

-- tokens_nats_v2 indexes
ALTER TABLE tokens_nats_v2
    ADD INDEX idx_name tokenbf_v1(10240, 3, 0) (token_name) GRANULARITY 4,
    ADD INDEX idx_symbol set(10000) (token_symbol) GRANULARITY 4,
    ADD INDEX idx_launchpad set(20) (launchpad) GRANULARITY 8;

-- enriched_trades indexes
ALTER TABLE enriched_trades
    ADD INDEX idx_is_sniper set(2) (is_sniper) GRANULARITY 8,
    ADD INDEX idx_is_insider set(2) (is_insider) GRANULARITY 8,
    ADD INDEX idx_is_whale set(2) (is_whale) GRANULARITY 8;

-- Materialize all
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_wallet;
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_price;
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_trade_type;
ALTER TABLE trades_nats_v3 MATERIALIZE PROJECTION proj_by_wallet;
```

### 7.4 Enable Optimizations

```sql
-- Session/user level settings for optimal queries
SET query_plan_optimize_lazy_materialization = 1;
SET optimize_move_to_prewhere = 1;
SET enable_multiple_prewhere_read_steps = 1;
SET move_all_conditions_to_prewhere = 1;
SET use_uncompressed_cache = 1;
SET max_threads = 8;
```

---

## 8. AggregatingMergeTree: Avoiding Stale Data

### 8.1 The Problem: Stale Data from Unmerged Parts

**Root cause**: AggregatingMergeTree merges data **asynchronously in the background**. Until parts are merged, you have multiple intermediate aggregate states that haven't been combined.

```
Insert batch 1 → Part A (partial aggregate state)
Insert batch 2 → Part B (partial aggregate state)
Insert batch 3 → Part C (partial aggregate state)
...
Background merge (eventually) → Combined Part
```

**If you query without proper aggregation, you get wrong/stale results!**

### 8.2 Understanding the Merge Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUERY EXECUTION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Part A (unmerged)     Part B (unmerged)    Part C (unmerged)  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │ open_state_1 │     │ open_state_2 │     │ open_state_3 │    │
│  │ high: 100    │     │ high: 105    │     │ high: 95     │    │
│  │ low: 90      │     │ low: 92      │     │ low: 88      │    │
│  │ close_state_1│     │ close_state_2│     │ close_state_3│    │
│  │ volume: 1000 │     │ volume: 500  │     │ volume: 800  │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         │                   │                    │              │
│         └───────────────────┼────────────────────┘              │
│                             │                                    │
│                             ▼                                    │
│                    GROUP BY + Merge                             │
│                    ┌──────────────┐                             │
│                    │ argMinMerge  │ → Finds earliest open       │
│                    │ max(high)    │ → 105                       │
│                    │ min(low)     │ → 88                        │
│                    │ argMaxMerge  │ → Finds latest close        │
│                    │ sum(volume)  │ → 2300                      │
│                    └──────────────┘                             │
│                             │                                    │
│                             ▼                                    │
│                    CORRECT RESULT                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 The Solution: Always Use GROUP BY with `-Merge` Suffix

#### Wrong Query (Stale Data)
```sql
-- WRONG: Direct SELECT without aggregation
SELECT
    token_address,
    candle_timestamp,
    open,    -- This is AggregateFunction state, not actual value!
    high,
    low,
    close,
    volume
FROM ohlcv_1d
WHERE token_address = 'ABC123';
-- Returns multiple rows per candle (one per unmerged part)
-- Data appears "stale" or duplicated
```

#### Correct Query (Always Accurate)
```sql
-- CORRECT: Use GROUP BY with -Merge suffix
SELECT
    token_address,
    candle_timestamp,
    argMinMerge(open) AS open,      -- Merge the aggregate states
    max(high) AS high,               -- SimpleAggregateFunction uses normal function
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_1d
WHERE token_address = 'ABC123'
GROUP BY token_address, candle_timestamp  -- CRITICAL: Must GROUP BY the ORDER BY columns
ORDER BY candle_timestamp;
```

### 8.4 Query Pattern Comparison

| Approach | Query Pattern | Performance | Correctness |
|----------|--------------|-------------|-------------|
| Direct SELECT | `SELECT * FROM table` | Fast | **WRONG** (stale data) |
| FINAL | `SELECT * FROM table FINAL` | **Slow** (single-threaded) | Correct |
| **GROUP BY + Merge** | `SELECT ...Merge() GROUP BY` | **Fast** | **Correct** |
| View wrapper | `SELECT * FROM v_table` | Fast | Correct |

### 8.5 Why FINAL is Slow

```sql
-- FINAL forces single-threaded merge at query time
SELECT * FROM ohlcv_1d FINAL WHERE token_address = 'ABC123';
```

**Problems with FINAL:**
- Forces full scan and merge across all involved parts
- Single-threaded execution (can't parallelize)
- High CPU and memory for large partitions
- Performance degrades with more unmerged parts

**When FINAL is acceptable:**
- Small tables (< 1M rows)
- Infrequent queries
- After manual OPTIMIZE TABLE

### 8.6 Recommended Pattern: Query Views

Create views that always return correct results:

```sql
-- =============================================
-- OHLCV TABLE (AggregatingMergeTree)
-- =============================================
CREATE TABLE ohlcv_1d
(
    token_address String,
    candle_timestamp Date,
    open AggregateFunction(argMin, Decimal64(18), DateTime64(3)),
    high SimpleAggregateFunction(max, Decimal64(18)),
    low SimpleAggregateFunction(min, Decimal64(18)),
    close AggregateFunction(argMax, Decimal64(18), DateTime64(3)),
    volume SimpleAggregateFunction(sum, Decimal64(6)),
    trade_count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYear(candle_timestamp)
ORDER BY (token_address, candle_timestamp)
SETTINGS
    min_age_to_force_merge_seconds = 14400,  -- Force merge after 4 hours
    merge_with_ttl_timeout = 3600;

-- =============================================
-- QUERY VIEW (Always returns correct data)
-- =============================================
CREATE VIEW v_ohlcv_1d AS
SELECT
    token_address,
    candle_timestamp,
    argMinMerge(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count
FROM ohlcv_1d
GROUP BY token_address, candle_timestamp;

-- =============================================
-- API QUERIES (Use the view - always accurate)
-- =============================================

-- Single token chart
SELECT * FROM v_ohlcv_1d
WHERE token_address = 'ABC123'
  AND candle_timestamp >= today() - 30
ORDER BY candle_timestamp;

-- Multiple tokens
SELECT * FROM v_ohlcv_1d
WHERE token_address IN ('ABC', 'DEF', 'GHI')
  AND candle_timestamp >= today() - 7
ORDER BY token_address, candle_timestamp;
```

### 8.7 Alternative: ReplacingMergeTree for Simpler Queries

If the `-Merge` pattern is too complex, consider `ReplacingMergeTree`:

```sql
CREATE TABLE ohlcv_1d_replacing
(
    token_address String,
    candle_timestamp Date,
    -- Store final values, not aggregate states
    open Decimal64(18),
    high Decimal64(18),
    low Decimal64(18),
    close Decimal64(18),
    volume Decimal64(6),
    trade_count UInt64,
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYear(candle_timestamp)
ORDER BY (token_address, candle_timestamp);

-- MV computes final values before inserting
CREATE MATERIALIZED VIEW mv_ohlcv_1d_replacing TO ohlcv_1d_replacing AS
SELECT
    token_address,
    toDate(candle_timestamp) AS candle_timestamp,
    argMinMerge(open) AS open,
    max(high) AS high,
    min(low) AS low,
    argMaxMerge(close) AS close,
    sum(volume) AS volume,
    sum(trade_count) AS trade_count,
    toUnixTimestamp64Milli(now64(3)) AS _version
FROM ohlcv_1h  -- Source table (already aggregated)
GROUP BY token_address, candle_timestamp;

-- Query with FINAL (simpler but still need dedup)
SELECT * FROM ohlcv_1d_replacing FINAL
WHERE token_address = 'ABC123';

-- Or use argMax pattern for dedup
SELECT
    token_address,
    candle_timestamp,
    argMax(open, _version) AS open,
    argMax(high, _version) AS high,
    argMax(low, _version) AS low,
    argMax(close, _version) AS close,
    argMax(volume, _version) AS volume
FROM ohlcv_1d_replacing
WHERE token_address = 'ABC123'
GROUP BY token_address, candle_timestamp;
```

### 8.8 Performance Optimizations

#### Force Background Merges (Maintenance Task)
```sql
-- Force merge for specific partition (run during low-traffic periods)
OPTIMIZE TABLE ohlcv_1d PARTITION '2025' FINAL;

-- Check merge progress and part count
SELECT
    table,
    partition,
    count() AS parts,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'ohlcv_1d' AND active
GROUP BY table, partition
ORDER BY parts DESC;

-- Alert if too many parts (should be < 100)
SELECT table, count() AS parts
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
HAVING parts > 100;
```

#### Tune Merge Settings
```sql
-- Make merges more aggressive
ALTER TABLE ohlcv_1d MODIFY SETTING
    -- Force merge of old parts
    min_age_to_force_merge_seconds = 14400,        -- 4 hours
    min_age_to_force_merge_on_partition_only = 1,  -- Only within partition

    -- Merge thresholds
    parts_to_throw_insert = 500,                   -- Error if too many parts
    parts_to_delay_insert = 300,                   -- Slow down inserts

    -- TTL merge settings
    merge_with_ttl_timeout = 3600;
```

#### Use Projections for Pre-Aggregated Results
```sql
-- Add projection that pre-computes merged results
ALTER TABLE ohlcv_1d ADD PROJECTION proj_merged (
    SELECT
        token_address,
        candle_timestamp,
        argMinMerge(open) AS open,
        max(high) AS high,
        min(low) AS low,
        argMaxMerge(close) AS close,
        sum(volume) AS volume,
        sum(trade_count) AS trade_count
    GROUP BY token_address, candle_timestamp
);

ALTER TABLE ohlcv_1d MATERIALIZE PROJECTION proj_merged;

-- Queries will automatically use projection when beneficial
```

### 8.9 Monitoring Merge Health

```sql
-- Check current merge activity
SELECT
    database,
    table,
    round(progress * 100, 2) AS progress_pct,
    round(total_size_bytes_compressed / 1e9, 2) AS size_gb,
    round(elapsed, 1) AS elapsed_sec
FROM system.merges
ORDER BY elapsed DESC;

-- Check merge history
SELECT
    event_date,
    table,
    count() AS merges,
    round(avg(duration_ms) / 1000, 1) AS avg_duration_sec,
    sum(read_rows) AS total_rows_merged
FROM system.part_log
WHERE event_type = 'MergeParts'
  AND event_date >= today() - 7
GROUP BY event_date, table
ORDER BY event_date DESC, merges DESC;

-- Parts per partition (identify problem areas)
SELECT
    table,
    partition,
    count() AS parts,
    min(modification_time) AS oldest_part,
    max(modification_time) AS newest_part
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table, partition
HAVING parts > 10
ORDER BY parts DESC;
```

### 8.10 Complete Click OHLCV Implementation with Views

```sql
-- =============================================
-- ALL TIMEFRAME TABLES
-- =============================================

-- 1-minute (from raw trades)
CREATE TABLE ohlcv_1m (...) ENGINE = AggregatingMergeTree() ...;
CREATE VIEW v_ohlcv_1m AS SELECT ... GROUP BY token_address, candle_timestamp;

-- 5-minute (from 1m)
CREATE TABLE ohlcv_5m (...) ENGINE = AggregatingMergeTree() ...;
CREATE VIEW v_ohlcv_5m AS SELECT ... GROUP BY token_address, candle_timestamp;

-- 1-hour (from 5m)
CREATE TABLE ohlcv_1h (...) ENGINE = AggregatingMergeTree() ...;
CREATE VIEW v_ohlcv_1h AS SELECT ... GROUP BY token_address, candle_timestamp;

-- 1-day (from 1h)
CREATE TABLE ohlcv_1d (...) ENGINE = AggregatingMergeTree() ...;
CREATE VIEW v_ohlcv_1d AS SELECT ... GROUP BY token_address, candle_timestamp;

-- =============================================
-- UNIFIED API VIEW
-- =============================================
CREATE VIEW v_ohlcv_all AS
SELECT '1m' AS timeframe, * FROM v_ohlcv_1m
UNION ALL
SELECT '5m' AS timeframe, * FROM v_ohlcv_5m
UNION ALL
SELECT '1h' AS timeframe, * FROM v_ohlcv_1h
UNION ALL
SELECT '1d' AS timeframe, * FROM v_ohlcv_1d;

-- =============================================
-- API USAGE
-- =============================================

-- Chart data for any timeframe
SELECT
    candle_timestamp,
    open, high, low, close, volume
FROM v_ohlcv_all
WHERE token_address = 'ABC123'
  AND timeframe = '1h'
  AND candle_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY candle_timestamp;
```

### 8.11 Key Takeaways

1. **Never query AggregatingMergeTree directly** without GROUP BY + `-Merge` suffix
2. **Create views** that encapsulate the correct query pattern
3. **Avoid FINAL** except for small tables or maintenance
4. **Monitor part count** - should stay under 100 per table
5. **Force merges** during low-traffic periods with OPTIMIZE TABLE
6. **Consider ReplacingMergeTree** if the aggregate pattern is too complex

---

## 9. JOIN Optimization Best Practices

### 9.1 Golden Rule: Smaller Table on Right

For hash-based joins, ClickHouse builds an in-memory hash table from the right-hand side of the JOIN. Always place the smaller table on the right.

```sql
-- GOOD: Small lookup table on right
SELECT t.*, m.token_name
FROM trades_nats_v3 t
JOIN tokens_nats_v2 m ON t.token_address = m.token_address;

-- As of ClickHouse 24.12, the query planner automatically places
-- the smaller table on the right side for optimal performance
```

### 9.2 JOIN Algorithm Selection

| Algorithm | Memory | Speed | Best For |
|-----------|--------|-------|----------|
| `hash` | High | Fast | Small right table, fits in memory |
| `partial_merge` | Low | Medium | Large tables, memory constrained |
| `full_sorting_merge` | Low | Fast if sorted | Tables sorted by join key |
| `auto` | Adaptive | Varies | Default - tries hash, falls back |

```sql
-- Force specific algorithm
SET join_algorithm = 'hash';  -- Default, fast but memory-intensive
SET join_algorithm = 'partial_merge';  -- Low memory
SET join_algorithm = 'auto';  -- Adaptive (recommended)

-- Auto tries hash first, switches to partial_merge if memory exceeded
SET max_bytes_in_join = 1000000000;  -- 1GB limit for hash join
```

### 9.3 Use IN Instead of JOIN for Key Existence

```sql
-- SLOW: JOIN just to check existence
SELECT * FROM trades t
JOIN (SELECT DISTINCT token_address FROM hot_tokens) h
ON t.token_address = h.token_address;

-- FAST: IN checks HashSet, no data extraction needed
SELECT * FROM trades
WHERE token_address IN (SELECT token_address FROM hot_tokens);
```

### 9.4 Use Dictionaries Instead of JOINs

Dictionaries are stored in memory as key-value maps, ideal for replacing joins on static/slow-changing reference data.

```sql
-- Create dictionary for token metadata
CREATE DICTIONARY token_dict (
    token_address String,
    token_name String,
    token_symbol String,
    launchpad String
)
PRIMARY KEY token_address
SOURCE(CLICKHOUSE(TABLE 'tokens_nats_v2'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);  -- Refresh every 5-10 minutes

-- Use dictGet instead of JOIN (much faster)
SELECT
    tx_signature,
    token_address,
    trade_total_usd,
    dictGet('token_dict', 'token_name', token_address) AS token_name,
    dictGet('token_dict', 'token_symbol', token_address) AS symbol
FROM trades_nats_v3
WHERE tx_timestamp > now() - INTERVAL 1 HOUR;
```

### 9.5 Denormalize at Insert Time

JOINs are expensive - shift computational work from query time to insert time:

```sql
-- Enriched trades table (denormalized)
CREATE TABLE enriched_trades (
    tx_signature String,
    token_address String,
    token_name String,        -- Denormalized from tokens
    token_symbol String,      -- Denormalized from tokens
    trade_total_usd Decimal64(18),
    trader_wallet String,
    is_sniper UInt8,          -- Pre-computed classification
    tx_timestamp DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(tx_timestamp)
ORDER BY (token_address, tx_timestamp);

-- MV enriches during insert
CREATE MATERIALIZED VIEW mv_enrich_trades TO enriched_trades AS
SELECT
    t.tx_signature,
    t.token_address,
    dictGet('token_dict', 'token_name', t.token_address) AS token_name,
    dictGet('token_dict', 'token_symbol', t.token_address) AS token_symbol,
    t.trade_total_usd,
    t.trader_wallet,
    t.is_sniper,
    t.tx_timestamp
FROM trades_nats_v3 t;
```

### 9.6 ClickHouse 24.4+ JOIN Improvements

Major improvements delivered 180x speedup on INNER and LEFT JOINs through:
- **Predicate pushdown**: Filters pushed closer to data scan
- **Better query planning**: Automatic join reordering (25.9+)

---

## 10. Dictionary Best Practices

### 10.1 Dictionary Layout Selection

| Layout | Memory | Lookup Speed | Best For |
|--------|--------|--------------|----------|
| `FLAT` | High | O(1) | Small, dense integer keys (< 500K) |
| `HASHED` | Medium | O(1) | String keys, medium size |
| `RANGE_HASHED` | Medium | O(1) | Temporal data with validity ranges |
| `CACHE` | Configurable | O(1) | Large source, unpredictable access |
| `COMPLEX_KEY_HASHED` | Medium | O(1) | Composite keys |

```sql
-- FLAT: Best performance for small integer keys
CREATE DICTIONARY small_lookup (
    id UInt64,
    value String
)
PRIMARY KEY id
LAYOUT(FLAT())
...;

-- HASHED: Default for string keys
CREATE DICTIONARY token_dict (
    token_address String,
    token_name String
)
PRIMARY KEY token_address
LAYOUT(HASHED())
...;

-- CACHE: For large external sources
CREATE DICTIONARY large_external (
    id UInt64,
    data String
)
PRIMARY KEY id
LAYOUT(CACHE(SIZE_IN_CELLS 100000))
SOURCE(MYSQL(...))
LIFETIME(300);
```

### 10.2 Incremental Dictionary Updates

For large dictionaries, use incremental updates:

```sql
CREATE DICTIONARY token_dict (
    token_address String,
    token_name String,
    updated_at DateTime
)
PRIMARY KEY token_address
SOURCE(CLICKHOUSE(
    TABLE 'tokens_nats_v2'
    UPDATE_FIELD updated_at  -- Only fetch rows updated since last refresh
    UPDATE_LAG 60            -- Allow 60 second lag
))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 120);
```

### 10.3 Click-Specific Dictionaries

```sql
-- Wallet classification dictionary
CREATE DICTIONARY wallet_classifications (
    wallet_address String,
    is_kol UInt8,
    is_whale UInt8,
    is_bot UInt8,
    label String DEFAULT ''
)
PRIMARY KEY wallet_address
SOURCE(CLICKHOUSE(TABLE 'wallet_labels'))
LAYOUT(HASHED())
LIFETIME(MIN 3600 MAX 7200);  -- Refresh hourly

-- Launchpad mapping
CREATE DICTIONARY launchpad_dict (
    launchpad_id UInt8,
    name String,
    graduation_threshold Decimal64(18)
)
PRIMARY KEY launchpad_id
LAYOUT(FLAT())
LIFETIME(86400);  -- Daily refresh

-- Usage in queries
SELECT
    trade_total_usd,
    dictGet('wallet_classifications', 'label', trader_wallet) AS wallet_label,
    dictGet('wallet_classifications', 'is_kol', trader_wallet) AS is_kol
FROM trades_nats_v3;
```

---

## 11. Async Inserts Best Practices

### 11.1 When to Use Async Inserts

| Scenario | Use Async? | Reason |
|----------|------------|--------|
| High-frequency small inserts | ✅ Yes | Reduces part creation overhead |
| Real-time streaming (< 1000 rows/insert) | ✅ Yes | Server-side batching |
| Large batch inserts (> 100K rows) | ❌ No | Already efficient |
| Need immediate consistency | ❌ No | Async has latency |

### 11.2 Configuration

```sql
-- Enable async inserts
SET async_insert = 1;
SET wait_for_async_insert = 1;  -- CRITICAL: Always use 1 in production!

-- Buffer flush triggers
SET async_insert_max_data_size = 10000000;     -- 10MB buffer size
SET async_insert_busy_timeout_ms = 1000;        -- Flush after 1 second
SET async_insert_max_query_number = 450;        -- Flush after 450 queries
```

**Critical Warning**: Never use `wait_for_async_insert = 0` in production unless you have explicit monitoring for data loss. Teams have lost millions of records by optimizing for speed without this setting.

### 11.3 Adaptive Async Inserts (v24.2+)

ClickHouse 24.2+ resolves back pressure issues with adaptive async inserts:

```sql
-- Adaptive settings
SET async_insert_use_adaptive_busy_timeout = 1;
SET async_insert_busy_timeout_min_ms = 50;
SET async_insert_busy_timeout_max_ms = 1000;
```

### 11.4 Monitoring Async Inserts

```sql
-- Check async insert queue
SELECT
    query_id,
    bytes,
    query
FROM system.asynchronous_inserts
ORDER BY first_update DESC
LIMIT 20;

-- Monitor flush performance
SELECT
    event_time,
    query,
    written_rows,
    result_bytes,
    query_duration_ms
FROM system.query_log
WHERE query_kind = 'AsyncInsertFlush'
  AND event_time > now() - INTERVAL 1 HOUR
ORDER BY event_time DESC;
```

---

## 12. Memory Management

### 12.1 Key Memory Settings

| Setting | Default | Purpose |
|---------|---------|---------|
| `max_memory_usage` | 10GB | Per-query memory limit |
| `max_server_memory_usage` | Auto (90% RAM) | Total ClickHouse memory |
| `max_bytes_before_external_group_by` | 0 (disabled) | Spill GROUP BY to disk |
| `max_bytes_before_external_sort` | 0 (disabled) | Spill ORDER BY to disk |

### 12.2 Recommended Configuration

```sql
-- Per-query limits
SET max_memory_usage = 10000000000;  -- 10GB per query

-- Enable disk spilling for large aggregations
SET max_bytes_before_external_group_by = 5000000000;  -- 5GB
SET max_bytes_before_external_sort = 5000000000;      -- 5GB

-- Limit threads per query (reduces memory)
SET max_threads = 8;
```

### 12.3 Server-Level Settings (config.xml)

```xml
<clickhouse>
    <!-- Leave 10-25% for OS -->
    <max_server_memory_usage_to_ram_ratio>0.8</max_server_memory_usage_to_ram_ratio>

    <!-- Mark cache for index navigation -->
    <mark_cache_size>5368709120</mark_cache_size>  <!-- 5GB -->

    <!-- Uncompressed cache for hot data -->
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>  <!-- 8GB -->
</clickhouse>
```

### 12.4 Memory Usage Patterns

In a well-tuned system:
- 10-30% RAM under normal conditions
- 50-70% during large queries
- Remaining RAM used by OS page cache

### 12.5 Preventing OOM Kills

```sql
-- Monitor memory usage
SELECT
    formatReadableSize(memory_usage) AS current_memory,
    formatReadableSize(peak_memory_usage) AS peak_memory,
    query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10;

-- Kill memory-heavy queries
KILL QUERY WHERE memory_usage > 8000000000;  -- > 8GB
```

---

## 13. Query Cache

### 13.1 Query Result Cache (v23.1+)

```sql
-- Enable query cache
SET use_query_cache = 1;

-- Configure cache behavior
SET query_cache_ttl = 60;                     -- 60 second TTL
SET query_cache_share_between_users = 0;      -- User-isolated cache
SET query_cache_min_query_duration = 1000;    -- Only cache queries > 1s
SET query_cache_min_query_runs = 2;           -- Cache after 2nd execution

-- Force cache a specific query
SELECT * FROM trades_nats_v3
WHERE token_address = 'ABC123'
SETTINGS use_query_cache = 1;
```

### 13.2 Query Condition Cache (v25.5+)

Stores filter condition results at granule level - much more memory-efficient than result cache:

```sql
-- Enable condition cache (default ON in 25.5+)
SET use_query_condition_cache = 1;
SET query_condition_cache_size = 100000000;  -- 100MB

-- Performance impact: 0.481s → 0.037s (10x faster)
```

### 13.3 Mark Cache

Caches primary index marks for faster granule lookups:

```xml
<!-- config.xml -->
<mark_cache_size>5368709120</mark_cache_size>  <!-- 5GB recommended -->
```

### 13.4 Uncompressed Cache

Caches decompressed blocks for frequently accessed data:

```sql
SET use_uncompressed_cache = 1;
```

---

## 14. Partitioning Best Practices

### 14.1 Key Rules

1. **Low cardinality**: 100-1,000 partitions max
2. **Monthly is usually enough**: Daily creates too many parts
3. **Partitioning is for data management**, not query optimization

### 14.2 Partition Key Selection

| Data Volume | Recommended Partition |
|-------------|----------------------|
| < 10GB/day | Monthly (`toYYYYMM`) |
| 10-100GB/day | Weekly (`toMonday`) |
| > 100GB/day | Daily (`toYYYYMMDD`) |

```sql
-- GOOD: Monthly partitioning
CREATE TABLE trades (...)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(tx_timestamp)
ORDER BY (token_address, tx_timestamp);

-- BAD: Hourly creates too many partitions
PARTITION BY toStartOfHour(tx_timestamp)  -- Don't do this!
```

### 14.3 Avoiding TOO_MANY_PARTS Error

```sql
-- Check part count per partition
SELECT
    table,
    partition,
    count() AS parts,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active
GROUP BY table, partition
HAVING parts > 50
ORDER BY parts DESC;

-- If too many parts:
-- 1. Use async inserts (server-side batching)
-- 2. Increase batch sizes
-- 3. Use coarser partitioning
```

### 14.4 Partition Management

```sql
-- Drop old partitions (data lifecycle)
ALTER TABLE trades DROP PARTITION '202401';

-- Move partition to cold storage
ALTER TABLE trades MOVE PARTITION '202401' TO VOLUME 'cold';

-- Detach/attach for backup
ALTER TABLE trades DETACH PARTITION '202401';
-- Copy files from detached/
ALTER TABLE trades ATTACH PARTITION '202401';
```

---

## 15. Distributed Queries & Sharding

### 15.1 Sharding Key Selection

Choose keys that distribute data evenly:

```sql
-- GOOD: Hash of high-cardinality column
CREATE TABLE trades_distributed AS trades
ENGINE = Distributed(
    cluster_name,
    default,
    trades,
    sipHash64(token_address)  -- Even distribution
);

-- BAD: Low cardinality creates hotspots
sipHash64(trade_type)  -- Only 2 values!
```

### 15.2 Insert Best Practices

```sql
-- RECOMMENDED: Insert to local tables, not distributed
-- More control, better performance
INSERT INTO trades_local VALUES (...);

-- If inserting to distributed table, enable internal replication
-- (prevents duplicates across replicas)
<shard>
    <internal_replication>true</internal_replication>
    <replica>...</replica>
</shard>
```

### 15.3 Query Optimization

```sql
-- Push filters to shards (reduces data transfer)
SET distributed_push_down_limit = 1;
SET optimize_skip_unused_shards = 1;

-- For aggregations, aggregate on shards first
SET distributed_aggregation_memory_efficient = 1;
```

### 15.4 ClickHouse Cloud (SharedMergeTree)

ClickHouse Cloud uses SharedMergeTree which doesn't need traditional sharding:
- Data stored in shared object storage
- Compute nodes scale independently
- No manual shard management

---

## 16. Deduplication Strategies

### 16.1 Engine Comparison

| Engine | Dedup Timing | Query Approach | Best For |
|--------|--------------|----------------|----------|
| ReplacingMergeTree | Background merge | FINAL or argMax | Upserts, CDC |
| CollapsingMergeTree | Background merge | Sign column | Updates + deletes |
| VersionedCollapsingMergeTree | Background merge | Version + sign | Concurrent updates |

### 16.2 ReplacingMergeTree Pattern

```sql
CREATE TABLE positions (
    user_id String,
    token_address String,
    balance Decimal64(18),
    updated_at DateTime64(3),
    _version UInt64
)
ENGINE = ReplacingMergeTree(_version)
ORDER BY (user_id, token_address);

-- Query with deduplication
SELECT
    user_id,
    token_address,
    argMax(balance, _version) AS balance
FROM positions
GROUP BY user_id, token_address;

-- Or use FINAL (slower)
SELECT * FROM positions FINAL
WHERE user_id = 'abc123';
```

### 16.3 Avoiding FINAL Performance Issues

FINAL forces single-threaded merge - avoid on large tables:

```sql
-- Instead of FINAL, use high-cardinality partition key
-- FINAL runs in parallel across partitions
CREATE TABLE positions (...)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY sipHash64(user_id) % 32  -- 32 partitions
ORDER BY (user_id, token_address);

-- FINAL now parallelized across 32 partitions
SELECT * FROM positions FINAL WHERE user_id = 'abc123';
```

### 16.4 Refreshable Materialized Views

Deduplicate periodically instead of at query time:

```sql
-- Refreshable MV runs FINAL once, stores clean data
CREATE MATERIALIZED VIEW positions_clean
REFRESH EVERY 5 MINUTE
AS SELECT * FROM positions FINAL;

-- Queries hit clean table (no FINAL needed)
SELECT * FROM positions_clean WHERE user_id = 'abc123';
```

---

## 17. ClickHouse 2025 Features

### 17.1 Performance Highlights

| Feature | Version | Improvement |
|---------|---------|-------------|
| Lightweight Updates | 25.7 | 1,000x faster than ALTER UPDATE |
| New Parquet Reader | 25.8 | 1.81x faster queries |
| Query Condition Cache | 25.5 | 10x faster repeated queries |
| JOIN Reordering | 25.9 | Automatic multi-table optimization |
| count() Optimization | 25.x | 20-30% faster aggregations |

### 17.2 Lightweight SQL Updates (25.7+)

```sql
-- Now dramatically faster with patch-part mechanism
UPDATE trades SET is_processed = 1
WHERE tx_timestamp < today() - 30;

-- Benchmarked up to 4,000x faster than PostgreSQL
```

### 17.3 Vector Search (GA in 25.8)

```sql
-- Create table with vector index
CREATE TABLE embeddings (
    id UInt64,
    embedding Array(Float32),
    INDEX vec_idx embedding TYPE vector_similarity('hnsw', 'cosine')
)
ENGINE = MergeTree()
ORDER BY id;

-- Vector similarity search
SELECT id, L2Distance(embedding, [0.1, 0.2, ...]) AS distance
FROM embeddings
ORDER BY distance
LIMIT 10;
```

### 17.4 Native JSON Type (25.7+)

```sql
-- High-performance JSON with column pruning
CREATE TABLE events (
    id UInt64,
    data JSON  -- Native JSON type
)
ENGINE = MergeTree()
ORDER BY id;

-- Efficient sub-column access
SELECT data.user.name, data.event.type
FROM events;
```

### 17.5 New Functions (119 in 2025)

```sql
-- Notable additions
SELECT parseDateTime64('2025-01-20 10:30:00', 3, '%Y-%m-%d %H:%M:%S');
SELECT arrayFold((acc, x) -> acc + x, [1, 2, 3], 0);  -- Reduce
SELECT mapApply((k, v) -> (k, v * 2), map('a', 1, 'b', 2));
```

---

## 18. Compression Codec Best Practices

### 18.1 Codec Selection Guide

| Data Type | Recommended Codec | Reason |
|-----------|------------------|--------|
| DateTime | `Delta, LZ4` | Sequential timestamps compress well |
| Monotonic counters | `DoubleDelta, LZ4` | Constant increments |
| Float (metrics) | `Gorilla` | Designed for gauges |
| Strings (hot) | `LZ4` | Fast decompression |
| Strings (cold) | `ZSTD(3)` | Better compression |
| Low cardinality | `LowCardinality + LZ4` | Dictionary encoding |

### 18.2 Implementation

```sql
CREATE TABLE trades_optimized (
    tx_timestamp DateTime64(3) CODEC(Delta, LZ4),
    token_address LowCardinality(String) CODEC(LZ4),
    price_usd Decimal64(18) CODEC(Gorilla),
    trade_total_usd Decimal64(18) CODEC(Gorilla),
    trade_type Enum8('buy'=1, 'sell'=2) CODEC(LZ4),
    tx_signature String CODEC(ZSTD(3)),  -- Large, rarely accessed
    trader_wallet String CODEC(LZ4)
)
ENGINE = MergeTree()
...;
```

### 18.3 Checking Compression Ratios

```sql
SELECT
    column,
    formatReadableSize(sum(column_data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(column_data_uncompressed_bytes)) AS uncompressed,
    round(sum(column_data_uncompressed_bytes) / sum(column_data_compressed_bytes), 2) AS ratio
FROM system.parts_columns
WHERE table = 'trades_nats_v3' AND active
GROUP BY column
ORDER BY sum(column_data_uncompressed_bytes) DESC;
```

---

## 19. Nullable Antipattern

### 19.1 Why Avoid Nullable

Nullable columns create a separate UInt8 column to track NULL values:
- **~2x slower** queries
- **~10-15% more storage**
- Affects index efficiency

### 19.2 Better Alternatives

```sql
-- BAD: Nullable
CREATE TABLE trades (
    funding_source Nullable(String),
    funding_amount Nullable(Decimal64(18))
);

-- GOOD: Default values
CREATE TABLE trades (
    funding_source String DEFAULT '',
    funding_amount Decimal64(18) DEFAULT 0
);

-- GOOD: LowCardinality with empty string
CREATE TABLE trades (
    funding_source LowCardinality(String) DEFAULT ''
);
```

### 19.3 If You Must Use Nullable

```sql
-- Use assumeNotNull when you know value isn't NULL
SELECT assumeNotNull(funding_source)
FROM trades
WHERE funding_source IS NOT NULL;

-- LowCardinality(Nullable(String)) is slightly better than Nullable(String)
funding_source LowCardinality(Nullable(String))
```

---

## Sources

### Official ClickHouse Documentation & Blogs
- [ClickHouse: Chaining Materialized Views](https://clickhouse.com/blog/chaining-materialized-views)
- [ClickHouse: Using Materialized Views](https://clickhouse.com/blog/using-materialized-views-in-clickhouse)
- [ClickHouse: Lazy Materialization](https://clickhouse.com/blog/clickhouse-gets-lazier-and-faster-introducing-lazy-materialization)
- [ClickHouse: PREWHERE Optimization](https://clickhouse.com/docs/optimize/prewhere)
- [ClickHouse: Cascading Materialized Views](https://clickhouse.com/docs/guides/developer/cascading-materialized-views)
- [ClickHouse: Data Skipping Indexes](https://clickhouse.com/docs/best-practices/use-data-skipping-indices-where-appropriate)
- [ClickHouse: Build Real-Time Market Data App](https://clickhouse.com/blog/build-a-real-time-market-data-app-with-clickhouse-and-polygonio)
- [ClickHouse: AggregatingMergeTree docs](https://clickhouse.com/docs/engines/table-engines/mergetree-family/aggregatingmergetree)
- [ClickHouse: FINAL Query Optimization](https://www.oreateai.com/blog/clickhouse-final-query-performance-optimization-practices/af8f0a6df887b655bf5cde9504dc1e9f)
- [ClickHouse: Asynchronous Data Inserts](https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse)
- [ClickHouse: Async Inserts Documentation](https://clickhouse.com/docs/optimize/asynchronous-inserts)
- [ClickHouse: Query Cache Introduction](https://clickhouse.com/blog/introduction-to-the-clickhouse-query-cache-and-design)
- [ClickHouse: Query Condition Cache](https://clickhouse.com/blog/introducing-the-clickhouse-query-condition-cache)
- [ClickHouse: Choosing a Partitioning Key](https://clickhouse.com/docs/best-practices/choosing-a-partitioning-key)
- [ClickHouse: Minimize and Optimize JOINs](https://clickhouse.com/docs/best-practices/minimize-optimize-joins)
- [ClickHouse: Choosing the Right JOIN Algorithm](https://clickhouse.com/blog/clickhouse-fully-supports-joins-how-to-choose-the-right-algorithm-part5)
- [ClickHouse: MVs vs Projections](https://clickhouse.com/docs/managing-data/materialized-views-versus-projections)
- [ClickHouse: Compression in ClickHouse](https://clickhouse.com/docs/data-compression/compression-in-clickhouse)
- [ClickHouse: Avoid Nullable Columns](https://clickhouse.com/docs/optimize/avoid-nullable-columns)
- [ClickHouse: Dictionaries Documentation](https://clickhouse.com/docs/sql-reference/dictionaries)
- [ClickHouse: Deduplication Strategies](https://clickhouse.com/docs/guides/developer/deduplication)
- [ClickHouse: Table Shards and Replicas](https://clickhouse.com/docs/shards)
- [ClickHouse: Alexey's Favorite Features 2025](https://clickhouse.com/blog/alexey-favorite-features-2025)
- [ClickHouse: New Functions 2025](https://clickhouse.com/blog/new-functions-2025)
- [ClickHouse: Release 25.8](https://clickhouse.com/blog/clickhouse-release-25-08)
- [ClickHouse: Release 25.11](https://clickhouse.com/blog/clickhouse-release-25-11)
- [ClickHouse: 13 Deadly Sins](https://clickhouse.com/blog/common-getting-started-issues-with-clickhouse)

### Altinity Resources
- [Altinity: Skip Indexes Black Magic](https://altinity.com/blog/clickhouse-black-magic-skipping-indices)
- [Altinity: Materialized Views Illuminated](https://altinity.com/blog/clickhouse-materialized-views-illuminated-part-1)
- [Altinity: AggregatingMergeTree Guide](https://kb.altinity.com/engines/mergetree-table-engine-family/aggregatingmergetree/)
- [Altinity: Using Async Inserts](https://altinity.com/blog/using-async-inserts-for-peak-data-loading-rates-in-clickhouse)
- [Altinity: ReplacingMergeTree Explained](https://altinity.com/blog/clickhouse-replacingmergetree-explained-the-good-the-bad-and-the-ugly)
- [Altinity: Dictionaries Explained](https://medium.com/altinity/clickhouse-dictionaries-explained-77ceaed410d4)
- [Altinity: Memory Configuration Settings](https://kb.altinity.com/altinity-kb-setup-and-maintenance/altinity-kb-memory-configuration-settings/)
- [Altinity: JOIN Optimization Tricks](https://kb.altinity.com/altinity-kb-queries-and-syntax/joins/joins-tricks/)
- [Altinity: Rescuing ClickHouse from OOM Killer](https://altinity.com/blog/rescuing-clickhouse-from-the-linux-oom-killer)
- [Altinity: Caching Definitive Guide](https://altinity.com/blog/caching-in-clickhouse-the-definitive-guide-part-1)

### Tinybird Resources
- [Tinybird: ClickHouse JOIN Improvements](https://www.tinybird.co/blog/clickhouse-joins-improvements)
- [Tinybird: Optimize ClickHouse Cluster](https://www.tinybird.co/blog/optimize-clickhouse-cluster)
- [Tinybird: Deduplication Strategies](https://www.tinybird.co/docs/guides/deduplication-strategies.html)
- [Tinybird: ReplacingMergeTree Examples](https://www.tinybird.co/blog/clickhouse-replacingmergetree-example)
- [Tinybird: NULL Behavior with LowCardinality](https://www.tinybird.co/blog/tips-10-null-behavior-with-lowcardinality-columns)

### Instaclustr Resources
- [Instaclustr: ClickHouse Best Practices Infrastructure](https://www.instaclustr.com/blog/mastering-clickhouse-best-practices-infrastructure-and-operational-excellence/)
- [Instaclustr: ClickHouse Best Practices Part 2](https://www.instaclustr.com/blog/clickhouse-best-practices-part-2-scaling-data-management-and-optimization/)
- [Instaclustr: ClickHouse Upgrades Best Practices](https://www.instaclustr.com/blog/a-guide-to-clickhouse-upgrades-and-best-practices/)

### Other Resources
- [Medium: Using ClickHouse for Financial Charts](https://medium.com/mop-developers/using-clickhouse-for-charts-profits-ad6dc56abf67)
- [Medium: argMax vs FINAL Optimization](https://medium.com/insiderengineering/clickhouse-query-optimization-argmax-vs-final-50c710a1a7f3)
- [Medium: Understanding Partitioning](https://medium.com/@demsyiman/understanding-partitioning-in-clickhouse-avoiding-common-pitfalls-for-optimal-performance-89efc131a734)
- [CloudQuery: Six Months with ClickHouse](https://www.cloudquery.io/blog/six-months-with-clickhouse-at-cloudquery)
- [GlassFlow: AggregatingMergeTree Explained](https://www.glassflow.dev/blog/aggregatingmergetree-clickhouse)
- [GlassFlow: ReplacingMergeTree](https://www.glassflow.dev/blog/replacingmergetree)
- [GlassFlow: ClickHouse Query Optimization](https://www.glassflow.dev/blog/clickhouse-query-optimization)
- [ChistaDATA: Chained Materialized Views](https://chistadata.com/chained-materialized-views-clickhouse/)
- [ChistaDATA: Index Granularity Tuning](https://chistadata.com/clickhouse-performance-index-granularity/)
- [ChistaDATA: NULL Values Performance](https://chistadata.com/how-can-null-values-affect-clickhouse-performance/)
- [Highlight.io: ClickHouse Performance Optimization](https://www.highlight.io/blog/lw5-clickhouse-performance-optimization)
- [Rill: Data Modeling Guide for ClickHouse](https://www.rilldata.com/blog/data-modeling-guide-for-real-time-analytics-with-clickhouse)
- [Datazip: Deduplication in ClickHouse](https://datazip.io/blog/how-deduplication-in-clickhouse-works-avoiding-final-keyword-for-better-performance-clz8brdt5003q27xk9filzg9u)
