# Building Real-Time Solana DEX Analytics with ClickHouse: Part 6

## Production Patterns, Query Optimization & Monitoring

*Part 6 of a 6-part series on building production-grade analytics for Solana DEX trading*

---

## Series Overview

| Part | Topic | Status |
|------|-------|--------|
| 1 | [Architecture Overview](./BLOG_01_ARCHITECTURE_OVERVIEW.md) | Published |
| 2 | [Trade Enrichment & OHLCV Aggregation](./BLOG_02_TRADE_ENRICHMENT_OHLCV.md) | Published |
| 3 | [Token Metrics & Rolling Windows](./BLOG_03_TOKEN_METRICS_ROLLING_WINDOWS.md) | Published |
| 4 | [Position Tracking & PnL Calculation](./BLOG_04_POSITION_TRACKING_PNL.md) | Published |
| 5 | [Discovery, Trending & Materialized Views](./BLOG_05_DISCOVERY_TRENDING_ANALYTICS.md) | Published |
| 6 | **Production Patterns & Query Optimization** | **You are here** |

---

## Introduction

Throughout this series, we've built a comprehensive real-time analytics system for Solana DEX trading. Now we address the critical question: **How do we make it production-ready?**

This final part covers:
- Storage optimization strategies for cost control
- Query optimization for sub-100ms dashboard responses
- Monitoring and alerting for operational excellence
- Common pitfalls and how to avoid them
- Capacity planning for growth

---

## Table of Contents

1. [Storage Optimization](#1-storage-optimization)
2. [Query Optimization Patterns](#2-query-optimization-patterns)
3. [Monitoring & Alerting](#3-monitoring--alerting)
4. [Deployment Best Practices](#4-deployment-best-practices)
5. [Common Pitfalls & Solutions](#5-common-pitfalls--solutions)
6. [Capacity Planning](#6-capacity-planning)
7. [Series Conclusion](#7-series-conclusion)

---

## 1. Storage Optimization

### The Storage Challenge

At production scale, storage costs can explode:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STORAGE GROWTH PROJECTION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  UNOPTIMIZED:                                                           │
│  ─────────────                                                          │
│  • enriched_trades: ~5M rows/day × 500 bytes = 2.5 GB/day              │
│  • token_metrics updates: ~100K tokens × 1KB × 60/min = 6 GB/day       │
│  • OHLCV (30 timeframes): ~500K candles/day × 200 bytes = 100 MB/day   │
│  • position_overview: ~50K updates/day × 500 bytes = 25 MB/day         │
│                                                                         │
│  Total unoptimized: ~9 GB/day = ~3.2 TB/year                           │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  OPTIMIZED (with strategies below):                                     │
│  ──────────────────────────────────                                     │
│  • Compression: 40% reduction                                          │
│  • TTL policies: 60% reduction for old data                            │
│  • Aggregation rollups: 90% reduction for historical                   │
│  • LowCardinality: 15% reduction on string columns                     │
│                                                                         │
│  Total optimized: ~1 TB/year (70% savings)                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Optimization 1: Compression Codecs

ClickHouse supports specialized compression per column type:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPRESSION CODEC SELECTION                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  COLUMN TYPE              RECOMMENDED CODEC           SAVINGS           │
│  ───────────              ─────────────────           ───────           │
│                                                                         │
│  Timestamps               DoubleDelta + ZSTD(1)       60-80%            │
│  (block_timestamp)        Sequential values compress  extremely well    │
│                                                                         │
│  Sequential IDs           Delta + ZSTD(1)             50-70%            │
│  (slot, block_number)     Delta stores differences                      │
│                                                                         │
│  Prices/Decimals          Gorilla + ZSTD(1)           40-60%            │
│  (price_sol, volume_usd)  Float-point algorithm                         │
│                                                                         │
│  Counts                   T64 + ZSTD(1)               30-50%            │
│  (trade_count, holders)   Integer optimization                          │
│                                                                         │
│  Short Strings            LZ4                         20-30%            │
│  (signature, address)     Fast compression                              │
│                                                                         │
│  Low-Cardinality          LowCardinality wrapper      70-90%            │
│  (dex, token_symbol)      Dictionary encoding                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Example: Optimized Table DDL**

```sql
CREATE TABLE enriched_trades_optimized (
    -- High-cardinality strings: ZSTD compression
    signature String CODEC(ZSTD(3)),
    trader String CODEC(ZSTD(3)),
    market String CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),

    -- Low-cardinality strings: Dictionary encoding
    dex LowCardinality(String),
    anchor_mint LowCardinality(String),  -- SOL, USDC, USDT only
    token_symbol LowCardinality(String),

    -- Timestamps: DoubleDelta for sequential values
    block_timestamp DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),
    server_timestamp DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),

    -- Decimals: Gorilla for floating-point patterns
    volume_sol Decimal128(18) CODEC(Gorilla, ZSTD(1)),
    volume_usd Decimal128(18) CODEC(Gorilla, ZSTD(1)),
    spot_price_sol Decimal128(18) CODEC(Gorilla, ZSTD(1)),
    spot_price_usd Decimal128(18) CODEC(Gorilla, ZSTD(1)),

    -- Small integers: No codec needed
    trade_direction UInt8,
    hops UInt8,
    hop_index UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature);
```

### Optimization 2: TTL Policies

Different data has different retention needs:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TTL STRATEGY BY TABLE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TABLE                    HOT         WARM        COLD        DELETE    │
│  ─────                    ───         ────        ────        ──────    │
│                                                                         │
│  enriched_trades          7 days      30 days     90 days     365 days  │
│  (raw trade data)         SSD         HDD         S3                    │
│                                                                         │
│  token_metrics            N/A (current state only, no TTL)              │
│  (latest per token)                                                     │
│                                                                         │
│  token_metrics_history    7 days      30 days     N/A         90 days   │
│  (historical snapshots)   SSD         HDD                               │
│                                                                         │
│  OHLCV (1s-30s)           7 days      N/A         N/A         7 days    │
│  OHLCV (1m-1h)            30 days     N/A         N/A         90 days   │
│  OHLCV (4h-1w)            N/A         N/A         N/A         365 days  │
│                                                                         │
│  position_overview        N/A (current state only, no TTL)              │
│  position_history         30 days     90 days     N/A         180 days  │
│                                                                         │
│  discovery                N/A (current state only, no TTL)              │
│  discovery_history        7 days      30 days     N/A         90 days   │
│                                                                         │
│  trending_* (aggregates)  30 days     90 days     365 days    N/A       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementation: Tiered Storage**

```sql
-- Table with tiered storage policy
CREATE TABLE enriched_trades (
    -- columns...
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature)
TTL
    block_timestamp + INTERVAL 7 DAY TO VOLUME 'warm',
    block_timestamp + INTERVAL 90 DAY TO VOLUME 'cold',
    block_timestamp + INTERVAL 365 DAY DELETE
SETTINGS storage_policy = 'tiered';
```

**Storage Policy Configuration (config.xml)**

```xml
<storage_configuration>
    <disks>
        <hot>
            <path>/var/lib/clickhouse/hot/</path>
        </hot>
        <warm>
            <path>/mnt/warm-storage/</path>
        </warm>
        <cold>
            <type>s3</type>
            <endpoint>https://s3.amazonaws.com/your-bucket/</endpoint>
            <access_key_id>***</access_key_id>
            <secret_access_key>***</secret_access_key>
        </cold>
    </disks>
    <policies>
        <tiered>
            <volumes>
                <hot><disk>hot</disk></hot>
                <warm><disk>warm</disk></warm>
                <cold><disk>cold</disk></cold>
            </volumes>
        </tiered>
    </policies>
</storage_configuration>
```

### Optimization 3: Aggregation Rollups

Raw data → Pre-aggregated summaries:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ROLLUP STRATEGY                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  RAW DATA (enriched_trades)                                             │
│  • 5 million rows/day                                                   │
│  • Individual trades with full detail                                   │
│  • Keep 90 days                                                         │
│                                                                         │
│         │                                                               │
│         │ Materialized View                                             │
│         ▼                                                               │
│                                                                         │
│  HOURLY ROLLUP (enriched_trades_hourly)                                 │
│  • ~50K rows/day (100x reduction)                                       │
│  • Per-token hourly aggregates                                          │
│  • Keep 365 days                                                        │
│                                                                         │
│         │                                                               │
│         │ Scheduled Job (daily)                                         │
│         ▼                                                               │
│                                                                         │
│  DAILY ROLLUP (enriched_trades_daily)                                   │
│  • ~5K rows/day (1000x reduction)                                       │
│  • Per-token daily aggregates                                           │
│  • Keep forever                                                         │
│                                                                         │
│  STORAGE COMPARISON:                                                    │
│  ───────────────────                                                    │
│  Raw (90 days):     450M rows × 500B = 225 GB                          │
│  Hourly (365 days): 18M rows × 200B = 3.6 GB                           │
│  Daily (forever):   1.8M rows × 150B = 270 MB                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Query Optimization Patterns

### The FINAL Modifier: When and How

For ReplacingMergeTree tables, queries need special handling:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FINAL MODIFIER PATTERNS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PROBLEM:                                                               │
│  ────────                                                               │
│  ReplacingMergeTree keeps duplicate rows until merge.                   │
│  Without FINAL, queries may return stale or duplicate data.             │
│                                                                         │
│  EXAMPLE (token_metrics):                                               │
│  ─────────────────────────                                              │
│                                                                         │
│  token_mint  │ price_usd │ last_updated                                 │
│  ────────────┼───────────┼──────────────                                │
│  ABC123      │ 1.50      │ 10:00:00  ← Old row                          │
│  ABC123      │ 1.75      │ 10:00:05  ← Current row                      │
│  ABC123      │ 1.60      │ 10:00:03  ← Intermediate (shouldn't appear)  │
│                                                                         │
│  SOLUTIONS:                                                             │
│  ──────────                                                             │
│                                                                         │
│  1. FINAL modifier (simple, slower):                                    │
│     SELECT * FROM token_metrics FINAL WHERE token_mint = 'ABC'          │
│     → Deduplicates at query time                                        │
│     → Works but scans more data                                         │
│                                                                         │
│  2. argMax pattern (faster for aggregations):                           │
│     SELECT                                                              │
│         token_mint,                                                     │
│         argMax(price_usd, last_updated) AS price_usd                    │
│     FROM token_metrics                                                  │
│     GROUP BY token_mint                                                 │
│     → Selects value from row with max last_updated                      │
│     → More efficient for large scans                                    │
│                                                                         │
│  3. Pre-materialized views (fastest):                                   │
│     Create a view that applies FINAL once                               │
│     Query the view without FINAL                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**When to Use Each Pattern:**

```sql
-- Pattern 1: FINAL for point queries
-- Good for: Single token lookup, small result sets
SELECT *
FROM token_metrics FINAL
WHERE token_mint = 'ABC123';

-- Pattern 2: argMax for aggregations
-- Good for: Leaderboards, top-N queries
SELECT
    token_mint,
    argMax(price_usd, last_updated) AS price_usd,
    argMax(market_cap_usd, last_updated) AS market_cap_usd,
    argMax(volume_24h_usd, last_updated) AS volume_24h_usd
FROM token_metrics
GROUP BY token_mint
ORDER BY volume_24h_usd DESC
LIMIT 100;

-- Pattern 3: Pre-materialized view
-- Good for: Frequently accessed current-state queries
CREATE VIEW v_token_metrics_current AS
SELECT * FROM token_metrics FINAL;

-- Query without FINAL
SELECT * FROM v_token_metrics_current
WHERE market_cap_usd > 1000000;
```

### Index Usage Patterns

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INDEX OPTIMIZATION                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PRIMARY KEY (ORDER BY) USAGE:                                          │
│  ─────────────────────────────                                          │
│                                                                         │
│  ORDER BY (token_mint, block_timestamp, signature)                      │
│                                                                         │
│  GOOD QUERIES (use primary key):                                        │
│  • WHERE token_mint = 'ABC'                     ✓ Uses first key       │
│  • WHERE token_mint = 'ABC'                                             │
│      AND block_timestamp > now() - INTERVAL 1 HOUR  ✓ Uses first 2 keys│
│                                                                         │
│  BAD QUERIES (skip primary key):                                        │
│  • WHERE trader = 'XYZ'                         ✗ Scans all data       │
│  • WHERE block_timestamp > now() - INTERVAL 1 HOUR  ✗ Second key only  │
│                                                                         │
│  SOLUTION: Secondary data skipping indexes                              │
│  ─────────────────────────────────────────                              │
│                                                                         │
│  INDEX idx_trader trader TYPE bloom_filter(0.01) GRANULARITY 4          │
│  INDEX idx_dex dex TYPE set(10) GRANULARITY 4                           │
│                                                                         │
│  QUERY PATTERNS BY TABLE:                                               │
│  ─────────────────────────                                              │
│                                                                         │
│  enriched_trades:                                                       │
│  • ORDER BY (token_mint, block_timestamp, signature)                    │
│  • Secondary: trader (bloom), dex (set)                                 │
│  • Common: "trades for token X in last hour"                            │
│  • Common: "trades by wallet Y"                                         │
│                                                                         │
│  position_overview:                                                     │
│  • ORDER BY (trader, token_mint)                                        │
│  • Common: "all positions for wallet X"                                 │
│  • Common: "position for wallet X + token Y"                            │
│                                                                         │
│  token_metrics:                                                         │
│  • ORDER BY (token_mint)                                                │
│  • Secondary: token_symbol (tokenbf_v1 for search)                      │
│  • Common: "metrics for token X"                                        │
│  • Common: "search tokens by symbol"                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Query Performance Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    QUERY PERFORMANCE CHECKLIST                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Before deploying a query to production:                                │
│                                                                         │
│  [ ] Check EXPLAIN output                                               │
│      EXPLAIN SELECT ... FROM table WHERE ...                            │
│      → Look for "Primary key condition" usage                          │
│                                                                         │
│  [ ] Measure query profile                                              │
│      SET send_logs_level = 'trace';                                     │
│      → Check rows read vs rows returned                                 │
│                                                                         │
│  [ ] Verify index usage                                                 │
│      SELECT * FROM system.query_log                                     │
│      WHERE query LIKE '%your_query%'                                    │
│      → Check read_rows, read_bytes                                      │
│                                                                         │
│  [ ] Test with FINAL if ReplacingMergeTree                              │
│      → Ensure correctness, then optimize if slow                        │
│                                                                         │
│  [ ] Check cardinality of filters                                       │
│      SELECT uniq(column) FROM table                                     │
│      → High cardinality on first ORDER BY column is ideal               │
│                                                                         │
│  [ ] Avoid SELECT *                                                     │
│      → Only select needed columns                                       │
│      → ClickHouse is columnar: fewer columns = faster                   │
│                                                                         │
│  [ ] Use PREWHERE for heavy filters                                     │
│      SELECT * FROM table PREWHERE heavy_filter                          │
│      → Filters before reading other columns                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dashboard Query Patterns

```sql
-- Pattern: Trending tokens (fast)
-- Uses: Pre-aggregated trending_token_holder_snapshots + token_metrics
SELECT
    tm.token_mint,
    tm.token_symbol,
    tm.price_usd,
    tm.market_cap_usd,
    tm.stats_1h_price_change_pct,
    tm.stats_24h_volume_usd
FROM token_metrics tm FINAL
WHERE tm.stats_24h_volume_usd > 10000
ORDER BY tm.stats_1h_price_change_pct DESC
LIMIT 50;

-- Pattern: Wallet portfolio (fast)
-- Uses: Primary key on (trader, token_mint)
SELECT
    token_mint,
    token_symbol,
    tokens_remaining,
    current_price_usd * tokens_remaining AS position_value_usd,
    realized_pnl_usd,
    unrealized_pnl_usd
FROM position_overview FINAL
WHERE trader = 'WALLET_ADDRESS'
  AND tokens_remaining > 0
ORDER BY position_value_usd DESC;

-- Pattern: Token trade history (fast)
-- Uses: Primary key on (token_mint, block_timestamp)
SELECT
    block_timestamp,
    trader,
    trade_direction,
    volume_usd,
    spot_price_usd
FROM enriched_trades
WHERE token_mint = 'TOKEN_ADDRESS'
  AND block_timestamp >= now() - INTERVAL 24 HOUR
ORDER BY block_timestamp DESC
LIMIT 100;

-- Pattern: Whale leaderboard (uses pre-aggregated table)
SELECT
    trader,
    total_trades,
    win_rate_pct,
    realized_pnl_usd,
    tier
FROM v_whale_leaderboard
ORDER BY realized_pnl_usd DESC
LIMIT 100;
```

---

## 3. Monitoring & Alerting

### Key Metrics to Monitor

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MONITORING DASHBOARD                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  INGESTION HEALTH:                                                      │
│  ─────────────────                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Metric                    │ Query                   │ Alert     │   │
│  ├───────────────────────────┼─────────────────────────┼───────────┤   │
│  │ Ingestion lag (seconds)   │ now() - max(timestamp)  │ > 60s     │   │
│  │ Events per second         │ count()/interval        │ < 100 eps │   │
│  │ Queue backlog             │ count(*) in queue table │ > 10000   │   │
│  │ MV errors                 │ system.errors           │ any       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  QUERY PERFORMANCE:                                                     │
│  ──────────────────                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Metric                    │ Source                  │ Alert     │   │
│  ├───────────────────────────┼─────────────────────────┼───────────┤   │
│  │ p95 query latency         │ system.query_log        │ > 500ms   │   │
│  │ Slow queries (>1s)        │ system.query_log        │ > 10/min  │   │
│  │ Failed queries            │ system.query_log        │ any       │   │
│  │ Concurrent queries        │ system.processes        │ > 50      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  STORAGE HEALTH:                                                        │
│  ───────────────                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Metric                    │ Source                  │ Alert     │   │
│  ├───────────────────────────┼─────────────────────────┼───────────┤   │
│  │ Disk usage %              │ system.disks            │ > 80%     │   │
│  │ Parts per table           │ system.parts            │ > 300     │   │
│  │ Merge queue size          │ system.mutations        │ > 100     │   │
│  │ Table growth rate         │ system.parts (delta)    │ unusual   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  RESOURCE USAGE:                                                        │
│  ───────────────                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Metric                    │ Source                  │ Alert     │   │
│  ├───────────────────────────┼─────────────────────────┼───────────┤   │
│  │ Memory usage              │ system.asynchronous_metrics │ > 80% │   │
│  │ CPU usage                 │ External monitoring     │ > 80%     │   │
│  │ Active connections        │ system.metrics          │ > 100     │   │
│  │ Replication lag           │ system.replicas         │ > 100 ops │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Monitoring Queries

```sql
-- 1. Ingestion lag by table
SELECT
    table,
    dateDiff('second', max_time, now()) AS lag_seconds
FROM (
    SELECT
        table,
        max(last_updated) AS max_time
    FROM (
        SELECT 'token_metrics' AS table, max(last_updated) AS last_updated
        FROM token_metrics FINAL
        UNION ALL
        SELECT 'position_overview', max(last_updated)
        FROM position_overview FINAL
        UNION ALL
        SELECT 'enriched_trades', max(block_timestamp)
        FROM enriched_trades
    )
    GROUP BY table
)
ORDER BY lag_seconds DESC;

-- 2. Query latency percentiles (last hour)
SELECT
    quantile(0.50)(query_duration_ms) AS p50_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms,
    count() AS total_queries
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
  AND query_kind = 'Select';

-- 3. Slow queries (>1 second)
SELECT
    query_start_time,
    query_duration_ms / 1000 AS duration_sec,
    read_rows,
    formatReadableSize(read_bytes) AS read_size,
    substring(query, 1, 200) AS query_preview
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY query_duration_ms DESC
LIMIT 20;

-- 4. Table storage sizes
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    formatReadableQuantity(sum(rows)) AS rows,
    count() AS parts
FROM system.parts
WHERE database = 'datastreams'
  AND active = 1
GROUP BY table
ORDER BY sum(bytes_on_disk) DESC;

-- 5. Parts count per table (merge health)
SELECT
    table,
    count() AS parts,
    sum(rows) AS total_rows,
    max(modification_time) AS last_modified
FROM system.parts
WHERE database = 'datastreams'
  AND active = 1
GROUP BY table
HAVING parts > 100
ORDER BY parts DESC;

-- 6. MV errors
SELECT
    name,
    value,
    last_error_time,
    last_error_message
FROM system.errors
WHERE name LIKE '%MaterializedView%'
  AND value > 0;
```

### Alerting Rules (Prometheus/Grafana)

```yaml
# Alert: High ingestion lag
- alert: ClickHouseIngestionLag
  expr: clickhouse_ingestion_lag_seconds > 60
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "ClickHouse ingestion lag is high"
    description: "Lag is {{ $value }}s for {{ $labels.table }}"

# Alert: Slow queries
- alert: ClickHouseSlowQueries
  expr: rate(clickhouse_slow_queries_total[5m]) > 0.1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High rate of slow queries"

# Alert: Disk space
- alert: ClickHouseDiskSpace
  expr: clickhouse_disk_used_ratio > 0.8
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "ClickHouse disk usage above 80%"

# Alert: Too many parts (merge backlog)
- alert: ClickHouseTooManyParts
  expr: clickhouse_table_parts > 300
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Table {{ $labels.table }} has too many parts"
```

---

## 4. Deployment Best Practices

### Deployment Order

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT ORDER                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ORDER MATTERS: Dependencies must be deployed first                     │
│                                                                         │
│  1. DATABASES                                                           │
│     └── CREATE DATABASE IF NOT EXISTS datastreams;                     │
│                                                                         │
│  2. TABLES (target tables for MVs)                                      │
│     ├── enriched_trades                                                │
│     ├── token_metrics                                                  │
│     ├── position_overview                                              │
│     └── ... all storage tables                                         │
│                                                                         │
│  3. QUEUE TABLES (NATS Engine)                                          │
│     ├── enriched_trades_queue                                          │
│     ├── token_metrics_queue                                            │
│     └── ... all queue tables                                           │
│                                                                         │
│  4. MATERIALIZED VIEWS (connect queues to tables)                       │
│     ├── enriched_trades_mv    [queue → table]                          │
│     ├── token_metrics_mv      [queue → table]                          │
│     ├── token_metrics_history_mv  [queue → history]                    │
│     └── ... all MVs                                                    │
│                                                                         │
│  5. VIEWS (API layer)                                                   │
│     ├── v_token_metrics_current                                        │
│     ├── v_whale_leaderboard                                            │
│     └── ... all views                                                  │
│                                                                         │
│  6. AGGREGATION TABLES + MVs                                            │
│     └── trending_*, rollup tables                                      │
│                                                                         │
│  CRITICAL: If MV is created before target table, data will be LOST!    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Migration Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEMA MIGRATION PATTERNS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ADDING A COLUMN:                                                       │
│  ────────────────                                                       │
│  ALTER TABLE token_metrics                                              │
│      ADD COLUMN new_field Decimal128(18) DEFAULT 0;                    │
│                                                                         │
│  -- Update queue table                                                  │
│  -- Update MV to include new column                                     │
│  -- No data migration needed (DEFAULT handles old rows)                 │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  RENAMING A COLUMN:                                                     │
│  ──────────────────                                                     │
│  ALTER TABLE token_metrics                                              │
│      RENAME COLUMN old_name TO new_name;                               │
│                                                                         │
│  -- Update queue table schema                                           │
│  -- Update MV SELECT list                                               │
│  -- Update all views referencing this column                            │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  CHANGING COLUMN TYPE:                                                  │
│  ─────────────────────                                                  │
│  -- ClickHouse allows some type changes in place                        │
│  ALTER TABLE token_metrics                                              │
│      MODIFY COLUMN field_name NewType;                                 │
│                                                                         │
│  -- For incompatible types: create new column, migrate, drop old       │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  CHANGING PRIMARY KEY (ORDER BY):                                       │
│  ─────────────────────────────────                                      │
│  -- Cannot change ORDER BY on existing table                            │
│  -- Must create new table and migrate data                              │
│                                                                         │
│  1. CREATE TABLE new_table (...) ORDER BY (new_key);                   │
│  2. INSERT INTO new_table SELECT * FROM old_table;                     │
│  3. RENAME TABLE old_table TO old_table_backup;                        │
│  4. RENAME TABLE new_table TO old_table;                               │
│  5. Update MVs to point to new table                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  For major schema changes, use parallel tables:                         │
│                                                                         │
│  ┌─────────────────┐    ┌─────────────────┐                            │
│  │   BLUE (live)   │    │  GREEN (new)    │                            │
│  │                 │    │                 │                            │
│  │ token_metrics   │    │ token_metrics_v2│                            │
│  │                 │    │                 │                            │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ │                            │
│  │ │ queue       │ │    │ │ queue_v2    │ │                            │
│  │ └─────────────┘ │    │ └─────────────┘ │                            │
│  │       │         │    │       │         │                            │
│  │       ▼         │    │       ▼         │                            │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ │                            │
│  │ │ mv          │ │    │ │ mv_v2       │ │                            │
│  │ └─────────────┘ │    │ └─────────────┘ │                            │
│  │       │         │    │       │         │                            │
│  │       ▼         │    │       ▼         │                            │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ │                            │
│  │ │ table       │ │    │ │ table_v2    │ │                            │
│  │ └─────────────┘ │    │ └─────────────┘ │                            │
│  └─────────────────┘    └─────────────────┘                            │
│          │                      │                                       │
│          │    SWITCH            │                                       │
│          ▼                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        API LAYER                                │   │
│  │                                                                 │   │
│  │   1. Deploy v2 tables + MVs                                    │   │
│  │   2. Both receive data (dual-write via NATS)                   │   │
│  │   3. Validate v2 data matches v1                               │   │
│  │   4. Switch API to v2                                          │   │
│  │   5. Monitor                                                   │   │
│  │   6. Drop v1 tables                                            │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Common Pitfalls & Solutions

### Pitfall 1: MV Created Before Target Table

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PITFALL: MV created before target table                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WRONG ORDER:                                                           │
│  ────────────                                                           │
│  CREATE TABLE queue (...) ENGINE = NATS ...;                           │
│  CREATE MATERIALIZED VIEW mv TO target AS SELECT ...;  ← ERROR!        │
│  CREATE TABLE target (...);                            ← Too late      │
│                                                                         │
│  RESULT: Data arrives but MV fails silently, data lost                 │
│                                                                         │
│  CORRECT ORDER:                                                         │
│  ──────────────                                                         │
│  CREATE TABLE target (...);                            ← First         │
│  CREATE TABLE queue (...) ENGINE = NATS ...;                           │
│  CREATE MATERIALIZED VIEW mv TO target AS SELECT ...;  ← Last          │
│                                                                         │
│  DETECTION:                                                             │
│  ──────────                                                             │
│  SELECT * FROM system.errors WHERE name LIKE '%MV%';                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pitfall 2: Forgetting FINAL on ReplacingMergeTree

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PITFALL: Missing FINAL on ReplacingMergeTree queries                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WRONG:                                                                 │
│  ──────                                                                 │
│  SELECT * FROM token_metrics WHERE token_mint = 'ABC';                 │
│  → May return multiple rows (old + new versions)                        │
│                                                                         │
│  CORRECT:                                                               │
│  ────────                                                               │
│  SELECT * FROM token_metrics FINAL WHERE token_mint = 'ABC';           │
│  → Returns only the latest version                                      │
│                                                                         │
│  FOR VIEWS:                                                             │
│  ──────────                                                             │
│  CREATE VIEW v_token_metrics AS                                         │
│  SELECT * FROM token_metrics FINAL;                                    │
│                                                                         │
│  → All queries against view are automatically deduplicated             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pitfall 3: Over-Partitioning

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PITFALL: Too many partitions                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WRONG:                                                                 │
│  ──────                                                                 │
│  PARTITION BY (toYYYYMMDD(timestamp), token_mint)                      │
│  → Creates: 365 days × 100K tokens = 36.5M partitions                  │
│  → ClickHouse limit: ~1000 partitions recommended                      │
│                                                                         │
│  CORRECT:                                                               │
│  ────────                                                               │
│  PARTITION BY toYYYYMM(timestamp)                                      │
│  → Creates: 12 partitions per year                                     │
│                                                                         │
│  RULE OF THUMB:                                                         │
│  • 1-50 partitions: Ideal                                              │
│  • 50-300 partitions: Acceptable                                       │
│  • 300+ partitions: Performance problems                               │
│                                                                         │
│  CHECK:                                                                 │
│  ──────                                                                 │
│  SELECT                                                                 │
│      table,                                                             │
│      count(DISTINCT partition) AS partition_count                      │
│  FROM system.parts                                                     │
│  WHERE active = 1                                                      │
│  GROUP BY table                                                        │
│  HAVING partition_count > 50;                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pitfall 4: Unbounded State Growth

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PITFALL: Tables without TTL growing forever                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  AFFECTED TABLES:                                                       │
│  ────────────────                                                       │
│  • token_metrics_history (no TTL = years of snapshots)                 │
│  • enriched_trades (no TTL = all trades forever)                       │
│  • discovery_history (no TTL = all state changes)                      │
│                                                                         │
│  SOLUTION:                                                              │
│  ─────────                                                              │
│  -- Add TTL to existing table                                          │
│  ALTER TABLE token_metrics_history                                     │
│      MODIFY TTL last_updated + INTERVAL 90 DAY DELETE;                 │
│                                                                         │
│  -- Force TTL cleanup                                                  │
│  ALTER TABLE token_metrics_history MATERIALIZE TTL;                    │
│                                                                         │
│  MONITORING:                                                            │
│  ───────────                                                            │
│  SELECT                                                                 │
│      table,                                                             │
│      formatReadableSize(sum(bytes_on_disk)) AS size,                   │
│      sum(rows) AS rows,                                                │
│      min(partition) AS oldest_partition                                │
│  FROM system.parts                                                     │
│  WHERE active = 1                                                      │
│  GROUP BY table                                                        │
│  ORDER BY sum(bytes_on_disk) DESC;                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pitfall 5: Incorrect Decimal Precision

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PITFALL: Decimal overflow or precision loss                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ISSUE:                                                                 │
│  ──────                                                                 │
│  Decimal128(18) can hold values up to 10^20                            │
│  Token supplies can exceed this for high-supply tokens                  │
│                                                                         │
│  SYMPTOMS:                                                              │
│  ─────────                                                              │
│  • Negative values appearing where positive expected                   │
│  • Zero values for non-zero inputs                                     │
│  • Insert failures                                                     │
│                                                                         │
│  SOLUTION:                                                              │
│  ─────────                                                              │
│  -- For large values (supply, total volume)                            │
│  Decimal256(18)  -- Huge range                                         │
│                                                                         │
│  -- For prices (smaller values, more precision)                        │
│  Decimal128(18)  -- Good balance                                       │
│                                                                         │
│  -- For percentages                                                    │
│  Decimal64(4)    -- 4 decimal places is enough                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Capacity Planning

### Sizing Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAPACITY PLANNING                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MEMORY:                                                                │
│  ───────                                                                │
│  Base requirement: 16 GB minimum for small deployments                 │
│  Production recommendation: 64-128 GB                                   │
│                                                                         │
│  Rule of thumb:                                                         │
│  • 1 GB per 100M rows in frequently queried tables                     │
│  • 2 GB per concurrent heavy query                                     │
│  • 10% overhead for merge operations                                   │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  STORAGE:                                                               │
│  ────────                                                               │
│  SSD (hot storage): Recent data (7-30 days)                            │
│  HDD (warm storage): Historical data (30-90 days)                      │
│  S3 (cold storage): Archive (90+ days)                                 │
│                                                                         │
│  Sizing formula:                                                        │
│  • enriched_trades: 500 bytes/row × events/day × retention_days        │
│  • token_metrics: 1 KB/row × unique_tokens (current state)             │
│  • OHLCV: 200 bytes/candle × timeframes × tokens × retention          │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  CPU:                                                                   │
│  ────                                                                   │
│  Ingestion: 1 core per 50K events/second                               │
│  Queries: 1 core per 10 concurrent heavy queries                       │
│  Merges: 2-4 cores reserved for background operations                  │
│                                                                         │
│  Production recommendation: 16-32 cores                                │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  NETWORK:                                                               │
│  ────────                                                               │
│  NATS → ClickHouse: 100-500 Mbps sustained for high volume             │
│  Client queries: Depends on result set sizes                           │
│  Replication (if clustered): Match ingestion bandwidth                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scaling Strategies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCALING STRATEGIES                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VERTICAL SCALING (single node):                                        │
│  ───────────────────────────────                                        │
│  Best for:                                                              │
│  • < 100K events/second                                                │
│  • < 10TB storage                                                      │
│  • < 50 concurrent queries                                             │
│                                                                         │
│  Approach:                                                              │
│  • Add more RAM for larger working sets                                │
│  • Add faster NVMe SSDs for hot storage                                │
│  • Add more cores for query parallelism                                │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  HORIZONTAL SCALING (cluster):                                          │
│  ─────────────────────────────                                          │
│  Best for:                                                              │
│  • > 100K events/second                                                │
│  • > 10TB storage                                                      │
│  • High availability requirements                                      │
│                                                                         │
│  Sharding strategies:                                                   │
│  • By token_mint: Good for token-centric queries                       │
│  • By time: Good for time-range queries                                │
│  • By trader: Good for portfolio queries                               │
│                                                                         │
│  Cluster topology:                                                      │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │    Shard 1              Shard 2              Shard 3           │    │
│  │  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐    │    │
│  │  │ Replica 1   │      │ Replica 1   │      │ Replica 1   │    │    │
│  │  └─────────────┘      └─────────────┘      └─────────────┘    │    │
│  │  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐    │    │
│  │  │ Replica 2   │      │ Replica 2   │      │ Replica 2   │    │    │
│  │  └─────────────┘      └─────────────┘      └─────────────┘    │    │
│  │                                                                │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  Replication factor: 2-3 for production                                │
│  Shard count: Start with 3, add as needed                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Series Conclusion

### What We Built

Over this 6-part series, we constructed a complete real-time analytics system:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE SYSTEM ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    SOLANA BLOCKCHAIN                             │   │
│  │         (Trades, Tokens, Pools, Transfers, Events)               │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    SOLANA INDEXER                                │   │
│  │   Raw events: trades.v2, transfers.v2, tokens, pools            │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │ NATS (RowBinary)                        │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    DATASTREAMS (Rustflow)                        │   │
│  │                                                                 │   │
│  │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │   │
│  │   │Trade Enricher │  │Token Metrics  │  │Position Track │      │   │
│  │   │    (Part 2)   │  │   (Part 3)    │  │   (Part 4)    │      │   │
│  │   └───────────────┘  └───────────────┘  └───────────────┘      │   │
│  │                                                                 │   │
│  │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │   │
│  │   │OHLCV Aggregate│  │Discovery Proc │  │Trending Aggs  │      │   │
│  │   │    (Part 2)   │  │   (Part 5)    │  │   (Part 5)    │      │   │
│  │   └───────────────┘  └───────────────┘  └───────────────┘      │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │ NATS (RowBinary)                        │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       CLICKHOUSE                                 │   │
│  │                                                                 │   │
│  │   Queue Tables (NATS Engine)                                    │   │
│  │        │                                                        │   │
│  │        ▼                                                        │   │
│  │   Materialized Views (routing + transformation)                 │   │
│  │        │                                                        │   │
│  │        ▼                                                        │   │
│  │   Storage Tables                                                │   │
│  │   ┌─────────────────────────────────────────────────────────┐  │   │
│  │   │ enriched_trades     │ MergeTree           │ Part 2      │  │   │
│  │   │ market_ohlcv        │ MergeTree           │ Part 2      │  │   │
│  │   │ token_ohlcv         │ MergeTree           │ Part 2      │  │   │
│  │   │ token_metrics       │ ReplacingMergeTree  │ Part 3      │  │   │
│  │   │ position_overview   │ ReplacingMergeTree  │ Part 4      │  │   │
│  │   │ discovery           │ ReplacingMergeTree  │ Part 5      │  │   │
│  │   │ trending_*          │ AggregatingMergeTree│ Part 5      │  │   │
│  │   │ whale_stats         │ ReplacingMergeTree  │ Part 5      │  │   │
│  │   └─────────────────────────────────────────────────────────┘  │   │
│  │        │                                                        │   │
│  │        ▼                                                        │   │
│  │   Views (API layer with FINAL, computed fields)                 │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       API / DASHBOARD                            │   │
│  │                                                                 │   │
│  │   • Token detail pages (metrics, trades, holders)               │   │
│  │   • Portfolio tracking (positions, PnL)                         │   │
│  │   • Discovery page (newly created, graduating)                  │   │
│  │   • Trending page (volume, launches, whales)                    │   │
│  │   • Charting (OHLCV across 15 timeframes)                       │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Learnings by Part

| Part | Topic | Key Takeaways |
|------|-------|---------------|
| 1 | Architecture | NATS + RowBinary + ClickHouse is the winning stack for real-time analytics |
| 2 | Trade Enrichment | Join multiple data sources at ingestion time, not query time |
| 3 | Token Metrics | Rolling windows with ring buffers enable constant-time updates |
| 4 | Position Tracking | FIFO cost basis and fee decomposition are essential for accurate PnL |
| 5 | Discovery & Trending | AggregatingMergeTree enables sub-100ms dashboard queries |
| 6 | Production | TTL, compression, and monitoring are the keys to operational success |

### Performance Targets Achieved

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE SUMMARY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  INGESTION:                                                             │
│  • Event throughput: 100K+ events/second                               │
│  • Ingestion lag: < 1 second (p95)                                     │
│  • Zero data loss with NATS JetStream                                  │
│                                                                         │
│  QUERY LATENCY:                                                         │
│  • Point queries (single token/wallet): < 50ms                         │
│  • Dashboard aggregations: < 100ms                                     │
│  • Complex analytics: < 500ms                                          │
│                                                                         │
│  STORAGE EFFICIENCY:                                                    │
│  • 70%+ compression with optimized codecs                              │
│  • 90%+ reduction for historical data via rollups                      │
│  • Tiered storage for cost optimization                                │
│                                                                         │
│  AVAILABILITY:                                                          │
│  • 99.9% uptime target with replication                                │
│  • Zero query failures from MV errors                                  │
│  • Graceful degradation under load                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### What's Next?

This architecture can be extended for:

1. **Multi-Chain Support** - Add Ethereum, Base, Arbitrum with same patterns
2. **Machine Learning** - Export data to feature stores for trading signals
3. **Real-Time Alerts** - Stream processing for price/volume alerts
4. **Social Analytics** - Correlate on-chain with social data

---

## Quick Reference Card

### ClickHouse Engine Selection

| Use Case | Engine | Example |
|----------|--------|---------|
| Append-only events | MergeTree | enriched_trades |
| Current state | ReplacingMergeTree | token_metrics, position_overview |
| Pre-aggregated metrics | AggregatingMergeTree | trending_launchpad_volume |
| Additive rollups | SummingMergeTree | daily_volume_rollups |

### NATS Subjects

| Subject | Description |
|---------|-------------|
| solana.trades.v2 | Raw swaps from indexer |
| solana.enriched_trades | Enriched trades |
| solana.token_metrics.v2 | Rolling token metrics |
| solana.position_overview | Wallet positions |
| solana.discovery.* | Token discovery by stage |
| solana.market_ohlcv.* | Market candles by timeframe |

### Optimization Checklist

- [ ] LowCardinality on low-cardinality string columns
- [ ] DoubleDelta + ZSTD on timestamp columns
- [ ] Gorilla + ZSTD on decimal/price columns
- [ ] TTL on all time-series tables
- [ ] FINAL or argMax on all ReplacingMergeTree queries
- [ ] Secondary indexes on frequently filtered columns
- [ ] Views wrapping FINAL for API layer
- [ ] Monitoring for ingestion lag, query latency, storage growth

---

*This concludes the 6-part series on Building Real-Time Solana DEX Analytics with ClickHouse. For questions or feedback, please refer to the documentation index.*

---

## Appendix: Complete Table Summary

| Table | Engine | Part | Purpose |
|-------|--------|------|---------|
| enriched_trades | MergeTree | 2 | Individual trades with enrichment |
| enriched_trades_by_trader | MergeTree | 2 | Trades indexed by trader |
| market_ohlcv | MergeTree | 2 | Market-level OHLCV candles |
| token_ohlcv | MergeTree | 2 | Token-level OHLCV candles |
| token_metrics | ReplacingMergeTree | 3 | Current token metrics |
| token_metrics_history | MergeTree | 3 | Historical token snapshots |
| position_overview | ReplacingMergeTree | 4 | Current wallet positions |
| position_history | MergeTree | 4 | Position change history |
| discovery | ReplacingMergeTree | 5 | Discovery token current state |
| discovery_history | MergeTree | 5 | Discovery state changes |
| trending_launchpad_volume_daily | AggregatingMergeTree | 5 | Daily volume by DEX |
| trending_launches_daily | AggregatingMergeTree | 5 | Daily launch counts |
| trending_graduations_daily | AggregatingMergeTree | 5 | Daily graduation counts |
| trending_hourly_intensity | AggregatingMergeTree | 5 | Hourly trading volume |
| trending_token_holder_snapshots | MergeTree | 5 | Hourly holder snapshots |
| trending_whale_stats | ReplacingMergeTree | 5 | Aggregated whale metrics |
