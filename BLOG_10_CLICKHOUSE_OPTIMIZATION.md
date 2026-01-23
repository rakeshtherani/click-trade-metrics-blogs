# ClickHouse Optimization Patterns for Real-Time Trading Analytics

## Overview

This comprehensive guide covers ClickHouse optimization patterns for high-throughput trading data pipelines. Based on production patterns from:

- **CryptoHouse** - ClickHouse's official Solana blockchain analytics platform
- **StockHouse** - Real-time stock and crypto market data (millions of updates/second)
- **sql.clickhouse.com** - Data pipeline patterns (S3Queue, Medallion architecture)

**Target workloads:**
- 100K+ trades/second ingestion
- Sub-second dashboard queries
- Real-time OHLCV candlestick generation
- Position tracking with PnL calculations
- Trending analytics with unique counts

---

# Part 1: Schema Design Patterns

## 1. CryptoHouse Core Schema (Production Reference)

CryptoHouse uses 7 core tables for Solana blockchain analytics. These patterns apply directly to trading platforms.

### 1.1 Blocks Table

```sql
CREATE TABLE solana.blocks
(
    block_hash String,
    block_timestamp DateTime64(6),      -- Microsecond precision
    height Int64,
    leader String,                       -- Validator pubkey
    leader_reward Decimal(38, 9),        -- 9 decimals for lamports
    previous_block_hash String,
    slot Int64,
    transaction_count Int64
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, slot)
ORDER BY (block_timestamp, slot, block_hash)
```

**Key Design Decisions:**
- `DateTime64(6)` - Microsecond precision for Solana's ~400ms block times
- `Decimal(38, 9)` - 9 decimal places matches SOL's lamport precision (1 SOL = 1B lamports)
- Primary Key on `(block_timestamp, slot)` - Time-first for range queries, slot for uniqueness

### 1.2 Transactions Table (Nested Array Patterns)

```sql
CREATE TABLE solana.transactions
(
    -- Nested account data as Array of Tuples
    accounts Array(Tuple(
        pubkey String,
        signer Bool,
        writable Bool
    )),

    -- Balance changes for all accounts in transaction
    balance_changes Array(Tuple(
        account String,
        before Decimal(38, 9),
        after Decimal(38, 9)
    )),

    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,                          -- JSON error or 'null'
    fee Decimal(38, 9),
    index Int64,                         -- Transaction index in block
    log_messages Array(String),          -- Program logs

    -- Token balance snapshots
    post_token_balances Array(Tuple(
        account_index Int64,
        mint String,
        owner String,
        amount Decimal(76, 38),          -- High precision for tokens
        decimals Int64
    )),
    pre_token_balances Array(Tuple(
        account_index Int64,
        mint String,
        owner String,
        amount Decimal(76, 38),
        decimals Int64
    )),

    recent_block_hash String,
    signature String,                    -- Transaction signature (unique ID)
    status String                        -- 'Success' or 'Fail'
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, signature)
```

**Key Design Decisions:**
- `Array(Tuple(...))` - Nested structures for related data (accounts, balances)
- `Decimal(76, 38)` - 38 decimal places for token amounts (handles any token precision)
- `log_messages Array(String)` - Preserves program execution logs for debugging
- `err String` - JSON string allows flexible error parsing

### 1.3 Accounts Table (Polymorphic Schema)

```sql
CREATE TABLE solana.accounts
(
    account_type String,

    -- Validator-specific fields
    authorized_voters Array(Tuple(authorized_voter String, epoch Int64)),
    authorized_withdrawer String,
    commission Int64,
    epoch_credits Array(Tuple(credits String, epoch Int64, previous_credits String)),
    votes Array(Tuple(confirmation_count Int64, slot Int64)),

    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),

    -- Account data (base64 encoded)
    data Array(Tuple(raw String, encoding String)),

    executable Bool,
    is_native Bool,
    lamports Decimal(38, 9),
    last_timestamp Array(Tuple(slot Int64, timestamp DateTime64(6))),

    -- Token account fields
    mint String,
    token_amount Decimal(38, 9),
    token_amount_decimals Int64,

    node_pubkey String,
    owner String,
    prior_voters Array(Tuple(authorized_pubkey String, epoch_of_last_authorized_switch Int64, target_epoch Int64)),
    program String,
    program_data String,
    pubkey String,
    rent_epoch Int64,
    retrieval_timestamp DateTime64(6),
    root_slot Int64,
    space Int64,
    state String,
    tx_signature String
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, pubkey, block_hash)
```

**Key Design Decisions:**
- Polymorphic schema - handles regular accounts, token accounts, and validator accounts
- `Array(Tuple(...))` for validator voting data (epoch credits, votes)
- ORDER BY includes `pubkey` for efficient account lookups

### 1.4 Token Transfers Table (generateULID Pattern)

```sql
CREATE TABLE solana.token_transfers
(
    authority String,
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    decimals Decimal(38, 9),
    destination String,
    fee Decimal(38, 9),
    fee_decimals Decimal(38, 9),
    memo String,
    mint String,
    mint_authority String,
    source String,
    transfer_type String,                -- 'transfer', 'transferChecked', etc.
    tx_signature String,
    value Decimal(38, 9),

    id String DEFAULT generateULID()     -- Auto-generated sortable unique ID
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, id)
```

**Key Pattern: `generateULID()`**
- Generates sortable unique IDs (better than UUID for time-series)
- 26 characters vs 36 for UUID
- Time-extractable prefix for debugging
- Use when natural key isn't available or unique enough

---

## 2. Data Types for Crypto/Financial Data

### 2.1 Precision Guidelines

| Data Type | Decimals | Use Case | Example |
|-----------|----------|----------|---------|
| `Decimal(38, 9)` | 9 | SOL amounts (lamports) | 1.234567890 SOL |
| `Decimal(38, 18)` | 18 | Token amounts (ERC-20 standard) | Handles most tokens |
| `Decimal(76, 38)` | 38 | Token amounts (any precision) | Handles SHIB, etc. |
| `Decimal(38, 2)` | 2 | USD display values | $1,234.56 |
| `DateTime64(6)` | microseconds | Block/tx timestamps | Solana's ~400ms blocks |
| `DateTime64(9)` | nanoseconds | High-frequency data | Order execution times |
| `DateTime64(3)` | milliseconds | General timestamps | Ingestion time |
| `Int64` | - | Slots, epochs, indices | Block slot numbers |
| `UInt64` | - | Counts, sequences | Trade sequence |
| `String` | - | Addresses, signatures | Base58 encoded keys |
| `Bool` | - | Flags | is_nft, signer, writable |

### 2.2 Why Decimal over Float?

```sql
-- WRONG: Float loses precision
CREATE TABLE bad_example (amount Float64)  -- 0.1 + 0.2 != 0.3

-- CORRECT: Decimal preserves exact values
CREATE TABLE good_example (amount Decimal(38, 18))  -- Exact arithmetic
```

Financial data requires exact decimal arithmetic. ClickHouse's Decimal type provides this.

### 2.3 Array(Tuple) for Nested Blockchain Data

```sql
-- Pattern: Nested structures without JSON overhead
balance_changes Array(Tuple(
    account String,
    before Decimal(38, 9),
    after Decimal(38, 9)
))

-- Access nested fields
SELECT
    balance_changes[1].account AS first_account,
    balance_changes[1].after - balance_changes[1].before AS change
FROM solana.transactions
```

### 2.4 LowCardinality for Categorical Data

```sql
-- Use LowCardinality for columns with < 10,000 unique values
token_symbol LowCardinality(String),  -- ~50K tokens (borderline)
dex LowCardinality(String),           -- ~20 DEXes
venue LowCardinality(String),         -- ~10 venues
exchange LowCardinality(String),      -- ~50 exchanges
transfer_type LowCardinality(String), -- ~10 types
```

**Benefits:**
- Dictionary encoding reduces storage 10-100x
- Faster GROUP BY operations
- Automatic optimization by ClickHouse

### 2.5 Enum8 for Fixed Categories

```sql
-- Enum8 uses exactly 1 byte (vs variable for LowCardinality)
CREATE TABLE trades (
    trade_direction Enum8('BUY' = 0, 'SELL' = 1),
    order_type Enum8(
        'market' = 1,
        'limit' = 2,
        'stop' = 3,
        'stop_limit' = 4,
        'trailing_stop' = 5
    ),
    order_status Enum8(
        'pending' = 0,
        'open' = 1,
        'filled' = 2,
        'partial' = 3,
        'cancelled' = 4,
        'rejected' = 5,
        'expired' = 6
    ),
    event_type Enum8(
        'trade' = 1,
        'quote' = 2,
        'order' = 3,
        'cancel' = 4,
        'fill' = 5,
        'partial_fill' = 6,
        'reject' = 7,
        'liquidation' = 8
    )
)
```

**Benefits of Enum8:**
- 1 byte storage (vs variable for LowCardinality)
- Compile-time validation of values
- Better compression
- Self-documenting schema

**When to use Enum8:**
- Fixed set of values (won't change frequently)
- < 256 possible values
- High-frequency columns in WHERE clauses

---

## 3. Codec Optimization

### 3.1 Codec Selection Guide

| Column Type | Recommended Codec | Compression | Speed |
|------------|-------------------|-------------|-------|
| Timestamps | `DoubleDelta, ZSTD(1)` | Excellent | Fast |
| Monotonic integers (slot, sequence) | `Delta, ZSTD(1)` | Excellent | Fast |
| Other integers | `T64, ZSTD(1)` | Good | Fast |
| Strings (addresses, signatures) | `ZSTD(3)` | Good | Medium |
| Decimals | `ZSTD(1)` | Good | Fast |
| Booleans | None or `ZSTD(1)` | N/A | Fast |
| LowCardinality | None (handled internally) | Excellent | Fast |

### 3.2 Applied Example

```sql
CREATE TABLE enriched_trades (
    -- Sortable unique ID
    id String DEFAULT generateULID(),

    -- High cardinality strings: ZSTD(3) for better compression
    signature String CODEC(ZSTD(3)),
    trader String CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),

    -- Monotonic integer: Delta encoding
    slot UInt64 CODEC(Delta, ZSTD(1)),

    -- Timestamps: DoubleDelta for sequential times
    block_timestamp DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),

    -- Decimals: ZSTD(1) is sufficient
    volume_sol Decimal128(18) CODEC(ZSTD(1)),
    volume_usd Decimal128(18) CODEC(ZSTD(1)),

    -- Categorical: LowCardinality handles compression
    dex LowCardinality(String),
    token_symbol LowCardinality(String)
)
```

### 3.3 Compression Ratios

| Column Type | Raw Size | Compressed | Ratio |
|-------------|----------|------------|-------|
| Timestamps (DoubleDelta) | 8 bytes | ~0.5 bytes | 16:1 |
| Addresses (ZSTD(3)) | 44 bytes | ~8 bytes | 5:1 |
| Decimals (ZSTD(1)) | 16 bytes | ~4 bytes | 4:1 |
| LowCardinality strings | Variable | ~2 bytes | 10:1+ |
| Monotonic integers (Delta) | 8 bytes | ~1 byte | 8:1 |

---

## 4. Table Engine Selection

### 4.1 Engine Decision Tree

```
Is data append-only with no updates?
├── Yes → MergeTree
│   └── Need simple counters? → SummingMergeTree
│   └── Need complex aggregates? → AggregatingMergeTree
└── No (updates/dedup needed) → ReplacingMergeTree
    └── Need versioning? → Add version column
```

### 4.2 ReplacingMergeTree for State Tables

**Use case:** Position overview, token metrics, account state

```sql
CREATE TABLE position_overview (
    trader String,
    token_mint String,
    holdings_token Decimal(38, 18),
    realized_pnl_sol Decimal(38, 18),
    last_updated DateTime64(3),
    -- Version for ReplacingMergeTree
    version UInt64 MATERIALIZED toUnixTimestamp64Milli(last_updated)
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/position_overview',
    '{replica}',
    version  -- Keep latest version
)
ORDER BY (trader, token_mint)
```

**Key points:**
- Always use `FINAL` in queries or handle dedup in application
- Version column ensures latest state wins during merge
- ORDER BY determines deduplication key

### 4.3 SummingMergeTree for Simple Counters

**Use case:** Daily volume aggregations, trade counts

```sql
CREATE TABLE trades_per_day (
    date Date,
    symbol LowCardinality(String),
    count Int64,
    volume Decimal(38, 18)
)
ENGINE = SummingMergeTree
ORDER BY (symbol, date)

-- MV to populate
CREATE MATERIALIZED VIEW trades_per_day_mv
TO trades_per_day AS
SELECT
    block_timestamp::Date AS date,
    token_symbol AS symbol,
    count() AS count,
    sum(volume_sol) AS volume
FROM trades
GROUP BY date, symbol
```

**Query (automatic summing on merge):**
```sql
SELECT date, symbol, sum(count) AS count, sum(volume) AS volume
FROM trades_per_day
GROUP BY date, symbol
```

**Limitation:** Cannot track unique counts accurately.

### 4.4 AggregatingMergeTree for Complex Aggregates

**Use case:** Unique counts, averages, OHLC aggregates

```sql
CREATE TABLE trending_analytics_daily (
    date Date,
    venue LowCardinality(String),

    -- Sum aggregates
    volume_sol AggregateFunction(sum, Decimal128(18)),
    trade_count AggregateFunction(sum, UInt64),

    -- Unique count aggregates (HyperLogLog)
    unique_tokens AggregateFunction(uniqCombined(14), String),
    unique_traders AggregateFunction(uniqCombined(14), String),

    -- Average aggregates
    avg_trade_size AggregateFunction(avg, Decimal128(18))
)
ENGINE = AggregatingMergeTree
ORDER BY (date, venue)

-- MV to populate
CREATE MATERIALIZED VIEW trending_analytics_daily_mv
TO trending_analytics_daily AS
SELECT
    toDate(block_timestamp) AS date,
    dex AS venue,
    sumState(volume_sol) AS volume_sol,
    sumState(toUInt64(1)) AS trade_count,
    uniqCombinedState(14)(token_mint) AS unique_tokens,
    uniqCombinedState(14)(trader) AS unique_traders,
    avgState(volume_sol) AS avg_trade_size
FROM enriched_trades
GROUP BY date, venue
```

**Query pattern:**
```sql
SELECT
    date,
    venue,
    sumMerge(volume_sol) AS volume_sol,
    sumMerge(trade_count) AS trade_count,
    uniqCombinedMerge(14)(unique_tokens) AS unique_tokens,
    uniqCombinedMerge(14)(unique_traders) AS unique_traders,
    avgMerge(avg_trade_size) AS avg_trade_size
FROM trending_analytics_daily
GROUP BY date, venue
```

**uniqCombined(14) explained:**
- HyperLogLog algorithm with 2^14 = 16384 registers
- ~1% error rate for unique counts
- Mergeable across partitions
- Memory efficient (~16KB per aggregate)

---

## 5. ORDER BY and Partitioning

### 5.1 ORDER BY Best Practices

**Rule 1: Most filtered column first**
```sql
-- If queries always filter by token_mint first
ORDER BY (token_mint, block_timestamp, signature)
```

**Rule 2: Cardinality order (low to high within filter group)**
```sql
-- venue has ~10 values, date has ~365/year
ORDER BY (venue, date)  -- Better for venue + date queries
```

**Rule 3: Time-bucketed ORDER BY for high-frequency data (StockHouse pattern)**
```sql
-- Groups quotes into 1-minute buckets on disk
CREATE TABLE quotes (
    sym LowCardinality(String),
    bp Float64,                -- Bid price
    bs UInt64,                 -- Bid size
    ap Float64,                -- Ask price
    as UInt64,                 -- Ask size
    t UInt64,                  -- Timestamp (milliseconds epoch)
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (sym, t - (t % 60000));  -- Round to 60-second (1 minute) buckets
```

**Benefits of time-bucketing:**
- Related data stored together on disk
- Faster range queries within time windows
- Better compression ratios
- Optimal for OHLCV candlestick queries

### 5.2 Partitioning Strategy

```sql
-- High-frequency data (trades): Daily partitions
PARTITION BY toYYYYMMDD(block_timestamp)

-- Medium-frequency data (OHLCV): Monthly partitions
PARTITION BY toYYYYMM(candle_time)

-- Low-frequency data (positions): No partitioning or monthly
PARTITION BY toYYYYMM(last_updated)
```

### 5.3 Partition Alignment Table

| Data Type | TTL | Partition By | Alignment |
|-----------|-----|--------------|-----------|
| Short OHLCV (1s-30s) | 7 days | Daily | 7 partitions dropped/week |
| Medium OHLCV (1m-4h) | 90 days | Monthly | 3 partitions dropped/quarter |
| Long OHLCV (1D-1W) | Forever | Monthly | No TTL |
| Trade history | 365 days | Daily | 365 partitions dropped/year |
| Token metrics history | 30 days | Daily | 30 partitions dropped/month |

---

# Part 2: Staging and Ingestion Patterns

## 6. Staging Pattern (Null Engine + MV)

### 6.1 Why Staging Tables?

When ingesting from external sources (Kafka, NATS, APIs), data often arrives as strings. The staging pattern:
1. Accepts raw string data (no parsing errors on insert)
2. Transforms via Materialized View
3. Inserts typed data into main table
4. Staging table stores nothing (Null engine)

### 6.2 Stage Transactions Example (CryptoHouse)

```sql
-- Staging table: accepts strings, stores nothing
CREATE TABLE solana.stage_transactions
(
    accounts String,                     -- JSON string
    balance_changes String,              -- JSON string
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,
    fee Decimal(38, 9),
    index Int64,
    log_messages String,                 -- JSON array as string
    pre_token_balances String,
    post_token_balances String,
    recent_block_hash String,
    signature String,
    status String
)
ENGINE = Null  -- Doesn't store data, just passes to MV
```

### 6.3 Transform MV

```sql
CREATE MATERIALIZED VIEW solana.stage_transactions_mv
TO solana.transactions
AS SELECT
    -- Cast JSON strings to typed Arrays
    CAST(accounts, 'Array(Tuple(String, Bool, Bool))') AS accounts,
    CAST(balance_changes, 'Array(Tuple(String, Decimal(38, 9), Decimal(38, 9)))') AS balance_changes,

    block_hash,
    block_slot,
    block_timestamp,
    compute_units_consumed,
    err,
    fee,
    index,

    -- Parse JSON array
    JSONExtract(log_messages, 'Array(String)') AS log_messages,

    CAST(pre_token_balances, 'Array(Tuple(Int64, String, String, Decimal(76, 38), Int64))') AS pre_token_balances,
    CAST(post_token_balances, 'Array(Tuple(Int64, String, String, Decimal(76, 38), Int64))') AS post_token_balances,

    recent_block_hash,
    signature,
    status
FROM solana.stage_transactions
```

### 6.4 Stage Accounts Example

```sql
CREATE TABLE solana.stage_accounts
(
    account_type String,
    authorized_voters String,            -- JSON string
    authorized_withdrawer String,
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    commission Int64,
    data String,                         -- JSON string
    epoch_credits String,                -- JSON string
    executable Bool,
    is_native Bool,
    lamports Decimal(38, 9),
    last_timestamp String,               -- JSON string
    mint String,
    node_pubkey String,
    owner String,
    prior_voters String,                 -- JSON string
    program String,
    program_data String,
    pubkey String,
    rent_epoch Int64,
    retrieval_timestamp DateTime64(6),
    root_slot Int64,
    space Int64,
    state String,
    token_amount Decimal(38, 9),
    token_amount_decimals Int64,
    tx_signature String,
    votes String                         -- JSON string
)
ENGINE = Null

CREATE MATERIALIZED VIEW solana.stage_accounts_mv
TO solana.accounts
AS SELECT
    account_type,
    CAST(authorized_voters, 'Array(Tuple(String, Int64))') AS authorized_voters,
    authorized_withdrawer,
    block_hash,
    block_slot,
    block_timestamp,
    commission,
    JSONExtract(concat('[', data, ']'), 'Array(Tuple(String, String))') AS data,
    CAST(epoch_credits, 'Array(Tuple(String, Int64, String))') AS epoch_credits,
    executable,
    is_native,
    lamports,
    CAST(last_timestamp, 'Array(Tuple(Int64, DateTime64(6)))') AS last_timestamp,
    mint,
    node_pubkey,
    owner,
    CAST(prior_voters, 'Array(Tuple(String, Int64, Int64))') AS prior_voters,
    program,
    program_data,
    pubkey,
    rent_epoch,
    retrieval_timestamp,
    root_slot,
    space,
    state,
    token_amount,
    token_amount_decimals,
    tx_signature,
    CAST(votes, 'Array(Tuple(Int64, Int64))') AS votes
FROM solana.stage_accounts
```

### 6.5 Generic Staging Pattern for Trading

```sql
-- Stage table for JSON ingestion
CREATE TABLE stage_trades (
    raw_data String
) ENGINE = Null;

-- Transform MV
CREATE MATERIALIZED VIEW stage_trades_mv TO trades AS
SELECT
    JSONExtractString(raw_data, 'signature') AS signature,
    JSONExtractString(raw_data, 'trader') AS trader,
    JSONExtractString(raw_data, 'symbol') AS symbol,
    JSONExtractString(raw_data, 'exchange') AS exchange,
    parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')) AS timestamp,
    JSONExtractFloat(raw_data, 'price')::Decimal(38, 18) AS price,
    JSONExtractFloat(raw_data, 'quantity')::Decimal(38, 18) AS quantity,
    JSONExtractString(raw_data, 'side') AS side
FROM stage_trades
WHERE isValidJSON(raw_data) = 1;  -- Filter invalid JSON
```

---

## 7. Filtering Patterns (arrayExists)

### 7.1 Filter Vote Transactions (CryptoHouse)

Solana has ~70% vote transactions. Filter them for analytics:

```sql
-- Destination table for non-voting transactions
CREATE TABLE solana.transactions_non_voting
(
    -- Same schema as transactions
    accounts Array(Tuple(pubkey String, signer Bool, writable Bool)),
    balance_changes Array(Tuple(account String, before Decimal(38, 9), after Decimal(38, 9))),
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,
    fee Decimal(38, 9),
    index Int64,
    log_messages Array(String),
    post_token_balances Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    pre_token_balances Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    recent_block_hash String,
    signature String,
    status String
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, signature)

-- MV with arrayExists filter
CREATE MATERIALIZED VIEW solana.transactions_non_voting_mv
TO solana.transactions_non_voting
AS SELECT *
FROM solana.transactions
WHERE NOT arrayExists(
    account -> account.1 = 'Vote111111111111111111111111111111111111111',
    accounts
)
```

**Key Pattern: `arrayExists` with Lambda**
```sql
-- Check if any account matches condition
arrayExists(
    account -> account.1 = 'Vote111111111111111111111111111111111111111',
    accounts
)

-- Returns true if Vote program is in accounts array
```

### 7.2 Filter by Program ID

```sql
-- Find transactions involving a specific program
SELECT *
FROM solana.transactions
WHERE arrayExists(
    account -> account.1 = 'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA',
    accounts
)
AND block_timestamp >= now() - INTERVAL 1 DAY
```

---

## 8. S3Queue Engine for Continuous Ingestion

### 8.1 Basic S3Queue Setup

```sql
-- Continuous ingestion from S3/GCS without external orchestration
CREATE TABLE trades_queue (
    data String
)
ENGINE = S3Queue(
    'https://storage.googleapis.com/bucket/trades/*.parquet',
    'HMAC_KEY',
    'HMAC_SECRET',
    'Parquet'
)
SETTINGS
    mode = 'ordered',                          -- Process files in order
    s3queue_processing_threads_num = 10;       -- Parallel processing
```

### 8.2 S3Queue with Advanced Settings

```sql
CREATE TABLE historical_trades_queue (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    side LowCardinality(String)
)
ENGINE = S3Queue(
    's3://bucket/trades/{YYYY}/{MM}/{DD}/*.parquet',
    'AWS_KEY',
    'AWS_SECRET',
    'Parquet'
)
SETTINGS
    mode = 'unordered',                        -- Process files in any order (faster)
    s3queue_polling_min_timeout_ms = 1800000,  -- 30 min minimum poll interval
    s3queue_polling_max_timeout_ms = 2400000,  -- 40 min maximum poll interval
    s3queue_tracked_files_limit = 2000,        -- Track up to 2000 processed files
    s3queue_buckets = 30;                      -- Parallel processing buckets
```

### 8.3 S3Queue Settings Reference

| Setting | Value | Purpose |
|---------|-------|---------|
| `mode` | 'ordered' | Process files in lexicographic order |
| `mode` | 'unordered' | Process files in any order (faster) |
| `s3queue_polling_min_timeout_ms` | 1800000 | Min wait between polls (30 min) |
| `s3queue_polling_max_timeout_ms` | 2400000 | Max wait between polls (40 min) |
| `s3queue_tracked_files_limit` | 2000 | Max files to track as processed |
| `s3queue_buckets` | 30 | Parallel processing buckets |
| `s3queue_processing_threads_num` | 10 | Parallel processing threads |

### 8.4 Daily Queue Rotation Pattern

```sql
-- Create new queue for today (prevents unbounded file tracking)
CREATE TABLE trades_queue_${today} (...)
ENGINE = S3Queue('s3://bucket/trades/${today}/*.parquet', ...)
SETTINGS mode = 'ordered';

-- Route to main table via MV
CREATE MATERIALIZED VIEW trades_queue_${today}_mv
TO trades AS
SELECT * FROM trades_queue_${today};

-- Drop yesterday's queue (cleanup)
DROP TABLE IF EXISTS trades_queue_${yesterday};
DROP VIEW IF EXISTS trades_queue_${yesterday}_mv;
```

---

## 9. Medallion Architecture (Bronze → Silver → Gold)

### 9.1 Bronze Layer (Raw Ingestion)

```sql
-- Raw trades from WebSocket/Kafka (accepts everything)
CREATE TABLE trades_bronze (
    raw_data String,
    _file LowCardinality(String),
    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = MergeTree
ORDER BY ingested_at;

-- MV from S3Queue to Bronze
CREATE MATERIALIZED VIEW trades_bronze_mv TO trades_bronze AS
SELECT
    data AS raw_data,
    _file
FROM trades_queue
WHERE isValidJSON(data) = 1;        -- Only valid JSON
```

### 9.2 Silver Layer (Cleaned & Validated)

```sql
-- Parsed and validated trades
CREATE TABLE trades_silver (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    side LowCardinality(String),
    trade_id String,
    dedup_hash UInt64 MATERIALIZED cityHash64(symbol, exchange, trade_id, toString(timestamp))
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, dedup_hash)
PARTITION BY toYYYYMM(timestamp);

-- MV from Bronze to Silver (with validation)
CREATE MATERIALIZED VIEW trades_silver_mv TO trades_silver AS
SELECT
    parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')) AS timestamp,
    JSONExtractString(raw_data, 'symbol') AS symbol,
    JSONExtractString(raw_data, 'exchange') AS exchange,
    JSONExtractFloat(raw_data, 'price')::Decimal(38, 18) AS price,
    JSONExtractFloat(raw_data, 'quantity')::Decimal(38, 18) AS quantity,
    JSONExtractString(raw_data, 'side') AS side,
    JSONExtractString(raw_data, 'trade_id') AS trade_id
FROM trades_bronze
WHERE price > 0 AND quantity > 0;   -- Basic validation
```

### 9.3 Gold Layer (Aggregates)

```sql
-- Pre-aggregated OHLCV for dashboards
CREATE TABLE ohlcv_1m_gold (
    timestamp DateTime,
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    open Decimal(38, 18),
    high Decimal(38, 18),
    low Decimal(38, 18),
    close Decimal(38, 18),
    volume Decimal(38, 18),
    trade_count UInt64
)
ENGINE = SummingMergeTree
ORDER BY (symbol, exchange, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- MV from Silver to Gold
CREATE MATERIALIZED VIEW ohlcv_1m_gold_mv TO ohlcv_1m_gold AS
SELECT
    toStartOfMinute(timestamp) AS timestamp,
    symbol,
    exchange,
    argMin(price, timestamp) AS open,
    max(price) AS high,
    min(price) AS low,
    argMax(price, timestamp) AS close,
    sum(quantity) AS volume,
    count() AS trade_count
FROM trades_silver
GROUP BY timestamp, symbol, exchange;
```

---

## 10. Dead Letter Queue (DLQ) Pattern

### 10.1 DLQ Table

```sql
-- Store records that failed validation
CREATE TABLE trades_dlq (
    raw_data String,
    error_reason LowCardinality(String),
    ingested_at DateTime64(3) DEFAULT now64(3),
    _file LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY ingested_at
TTL ingested_at + INTERVAL 30 DAY;   -- Auto-cleanup after 30 days
```

### 10.2 Route Bad Records to DLQ

```sql
-- Good records → Silver
CREATE MATERIALIZED VIEW trades_silver_mv TO trades_silver AS
SELECT ...
FROM trades_bronze
WHERE price > 0
    AND quantity > 0
    AND isValidJSON(raw_data) = 1
    AND abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) < 86400;

-- Bad records → DLQ (inverted conditions)
CREATE MATERIALIZED VIEW trades_dlq_mv TO trades_dlq AS
SELECT
    raw_data,
    multiIf(
        isValidJSON(raw_data) = 0, 'invalid_json',
        JSONExtractFloat(raw_data, 'price') <= 0, 'invalid_price',
        JSONExtractFloat(raw_data, 'quantity') <= 0, 'invalid_quantity',
        abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) >= 86400, 'stale_timestamp',
        'unknown'
    ) AS error_reason,
    _file
FROM trades_bronze
WHERE price <= 0
    OR quantity <= 0
    OR isValidJSON(raw_data) = 0
    OR abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) >= 86400;
```

---

## 11. High-Throughput Ingestion (StockHouse Go Ingester)

### 11.1 Batch Insert Pattern

```go
// INSERT statements for each table type
"INSERT INTO %s.quotes (sym,bx,bp,bs,ax,ap,as,c,i,t,q,z) VALUES"
"INSERT INTO %s.trades (sym,i,x,p,s,c,t,q,z,trfi,trft) VALUES"
"INSERT INTO %s.crypto_quotes (pair,bp,bs,ap,as,t,x,r) VALUES"
"INSERT INTO %s.crypto_trades (pair,p,t,s,c,i,x,r) VALUES"
"INSERT INTO %s.stock_fmv (sym,fmv,t) VALUES"
```

### 11.2 Optimal Ingester Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| Channel capacity | 200,000 | Buffer for incoming WebSocket messages |
| Max rows per batch | 50,000 | Optimal batch size for inserts |
| Max flush interval | 1 second | Maximum wait before flushing |
| Insert timeout | 10 seconds | Per-batch timeout |
| Backpressure threshold | 95% (190,000) | When to start shedding load |
| Shedding rate | 1,000 items | Drop count when threshold exceeded |

### 11.3 ClickHouse Client Configuration (Go)

```go
// Native protocol with LZ4 compression
clickhouse.Open(&clickhouse.Options{
    Addr: []string{host},
    Auth: clickhouse.Auth{
        Database: database,
        Username: username,
        Password: password,
    },
    Compression: &clickhouse.Compression{
        Method: clickhouse.CompressionLZ4,      // LZ4 for speed
    },
    TLS: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
    DialTimeout: 30 * time.Second,
})
```

### 11.4 Key Ingestion Optimizations

| Optimization | Implementation | Benefit |
|--------------|----------------|---------|
| Native protocol | clickhouse-go/v2 | Faster than HTTP |
| LZ4 compression | CompressionLZ4 | 3-4x compression, low CPU |
| Batch inserts | 50,000 rows | Optimal throughput |
| Async buffering | 200k channel | Handle bursts |
| Load shedding | Drop at 95% | Prevent OOM |
| 1-second flush | Max interval | Low latency |

---

# Part 3: Aggregation Patterns

## 12. Daily Block & Transaction Counts (CryptoHouse)

```sql
CREATE TABLE solana.block_txn_counts_by_day
(
    day Date,
    block_count AggregateFunction(uniqCombined(14), String),
    txn_count AggregateFunction(sum, Int64)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.block_txn_counts_by_day_mv
TO solana.block_txn_counts_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    uniqCombinedState(14)(block_hash) AS block_count,  -- HyperLogLog state
    sumState(transaction_count) AS txn_count           -- Sum state
FROM solana.blocks
GROUP BY day
```

**Query Pattern:**
```sql
SELECT
    day,
    uniqCombinedMerge(14)(block_count) AS block_count,
    sumMerge(txn_count) AS txn_count
FROM solana.block_txn_counts_by_day
GROUP BY day
ORDER BY day DESC
```

## 13. Daily Fees & Signer Counts

```sql
CREATE TABLE solana.daily_fees_by_day
(
    day DateTime,
    avg_fee_sol AggregateFunction(avg, Float64),
    fee_sol AggregateFunction(sum, Float64),
    signer_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.daily_fees_by_day_mv
TO solana.daily_fees_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    avgState(fee / 1e9) AS avg_fee_sol,              -- Convert lamports to SOL
    sumState(fee / 1e9) AS fee_sol,
    uniqCombinedState(14)(accounts[1].1) AS signer_count  -- First account is signer
FROM solana.transactions_non_voting
GROUP BY day
```

## 14. Token Transfer Metrics

```sql
CREATE TABLE solana.token_transfer_metrics_by_day
(
    day DateTime,
    transfer_count AggregateFunction(uniqCombined(14), String),
    sender_count AggregateFunction(uniqCombined(14), String),
    reciever_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.token_transfer_metrics_by_day_mv
TO solana.token_transfer_metrics_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    uniqCombinedState(14)(tx_signature) AS transfer_count,
    uniqCombinedState(14)(source) AS sender_count,
    uniqCombinedState(14)(destination) AS reciever_count
FROM solana.token_transfers
GROUP BY day
```

## 15. Multi-Dimensional Aggregations (By Type)

```sql
CREATE TABLE solana.token_transfer_metrics_by_type_by_day
(
    day DateTime,
    transfer_type LowCardinality(String),
    transfer_count AggregateFunction(uniqCombined(14), String),
    sender_count AggregateFunction(uniqCombined(14), String),
    reciever_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY (day, transfer_type)

CREATE MATERIALIZED VIEW solana.token_transfer_metrics_by_type_by_day_mv
TO solana.token_transfer_metrics_by_type_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    transfer_type,
    uniqCombinedState(14)(tx_signature) AS transfer_count,
    uniqCombinedState(14)(source) AS sender_count,
    uniqCombinedState(14)(destination) AS reciever_count
FROM solana.token_transfers
GROUP BY day, transfer_type
```

## 16. Top Token Creators

```sql
CREATE TABLE solana.top_token_creators_by_day
(
    day DateTime,
    creator_address String,
    token_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY (day, creator_address)

CREATE MATERIALIZED VIEW solana.top_token_creators_by_day_mv
TO solana.top_token_creators_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    creators[1].1 AS creator_address,  -- First creator's address
    uniqCombinedState(14)(mint) AS token_count
FROM solana.tokens
GROUP BY day, creator_address
```

---

# Part 4: OHLCV Candlestick Patterns (StockHouse)

## 17. Real-Time Market Data Schema

### 17.1 Crypto Trades Table

```sql
CREATE TABLE crypto_trades
(
    pair String,                          -- Trading pair (BTC-USD, ETH-USD)
    p Float64,                            -- Price
    t UInt64,                             -- Timestamp (milliseconds)
    s Float64,                            -- Size
    c Array(UInt8),                       -- Conditions
    i String,                             -- Trade ID
    x UInt8,                              -- Exchange ID
    r UInt64,                             -- Received timestamp
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (pair, t - (t % 60000), i);      -- Time-bucketed + trade ID
```

### 17.2 Crypto Quotes Table

```sql
CREATE TABLE crypto_quotes
(
    pair String,                          -- Trading pair
    bp Float64,                           -- Bid price
    bs Float64,                           -- Bid size
    ap Float64,                           -- Ask price
    as Float64,                           -- Ask size
    t UInt64,                             -- Timestamp (milliseconds)
    x UInt8,                              -- Exchange ID
    r UInt64,                             -- Received timestamp
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (pair, t - (t % 60000));
```

## 18. Daily Aggregation Tables (AggregatingMergeTree)

### 18.1 Crypto Trades Daily Aggregation

```sql
CREATE TABLE agg_crypto_trades_daily
(
    event_date Date,
    pair LowCardinality(String),
    open_price_state AggregateFunction(argMin, Float64, UInt64),   -- First price of day
    last_price_state AggregateFunction(argMax, Float64, UInt64),   -- Last price of day
    volume_state AggregateFunction(sum, Float64),                   -- Total volume
    latest_t_state AggregateFunction(max, UInt64)                   -- Latest timestamp
)
PARTITION BY toYYYYMM(event_date)         -- Monthly partitions
ORDER BY (event_date, pair);

CREATE MATERIALIZED VIEW mv_crypto_trades_daily
TO agg_crypto_trades_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    pair,
    argMinState(p, t) AS open_price_state,    -- Price at earliest timestamp
    argMaxState(p, t) AS last_price_state,    -- Price at latest timestamp
    sumState(s) AS volume_state,
    maxState(t) AS latest_t_state
FROM crypto_trades
GROUP BY event_date, pair;
```

**Key Pattern: argMinState/argMaxState for OHLC**
```sql
-- argMinState(value, timestamp) → value at MIN timestamp (OPEN price)
-- argMaxState(value, timestamp) → value at MAX timestamp (CLOSE price)
-- This is the correct way to get open/close prices!

-- Query the aggregated data:
SELECT
    event_date,
    pair,
    argMinMerge(open_price_state) AS open,
    argMaxMerge(last_price_state) AS close,
    sumMerge(volume_state) AS volume
FROM agg_crypto_trades_daily
GROUP BY event_date, pair
```

### 18.2 Crypto Quotes Daily Aggregation

```sql
CREATE TABLE agg_crypto_quotes_daily
(
    event_date Date,
    pair LowCardinality(String),
    bid_state AggregateFunction(argMax, Float64, UInt64),    -- Latest bid
    ask_state AggregateFunction(argMax, Float64, UInt64),    -- Latest ask
    latest_t_state AggregateFunction(max, UInt64)
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, pair);

CREATE MATERIALIZED VIEW mv_crypto_quotes_daily
TO agg_crypto_quotes_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    pair,
    argMaxState(bp, t) AS bid_state,          -- Bid at latest timestamp
    argMaxState(ap, t) AS ask_state,          -- Ask at latest timestamp
    maxState(t) AS latest_t_state
FROM crypto_quotes
GROUP BY event_date, pair;
```

## 19. OHLC Candlestick Query Patterns

### 19.1 Generic OHLC Query Structure

```sql
-- Candlestick with configurable bucket size
SELECT
    toDateTime64(toStartOfInterval(
        fromUnixTimestamp64Milli(t, 'UTC'),
        INTERVAL {bucket_seconds} SECOND
    ), 3) AS timestamp,
    argMin(p, t) AS open,                 -- Price at first timestamp in bucket
    argMax(p, t) AS close,                -- Price at last timestamp in bucket
    max(p) AS high,                       -- Highest price in bucket
    min(p) AS low,                        -- Lowest price in bucket
    sum(s) AS volume                      -- Total volume in bucket
FROM crypto_trades
WHERE pair = {pair:String}
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL {window} MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC
```

### 19.2 Predefined Timeframe Queries

```sql
-- 15-second buckets, 10-minute window (live view)
SELECT
    toDateTime64(toStartOfInterval(fromUnixTimestamp64Milli(t, 'UTC'), INTERVAL 15 SECOND), 3) AS timestamp,
    argMin(p, t) AS open,
    argMax(p, t) AS close,
    max(p) AS high,
    min(p) AS low
FROM crypto_trades
WHERE pair = 'BTC-USD'
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL 10 MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC;

-- 1-minute buckets, 30-minute window
SELECT
    toDateTime64(toStartOfInterval(fromUnixTimestamp64Milli(t, 'UTC'), INTERVAL 1 MINUTE), 3) AS timestamp,
    argMin(p, t) AS open,
    argMax(p, t) AS close,
    max(p) AS high,
    min(p) AS low
FROM crypto_trades
WHERE pair = 'BTC-USD'
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL 30 MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC;
```

### 19.3 Live Data Query with Aggregated Tables

```sql
-- Combine daily aggregates with bid/ask spread
WITH trades AS (
    SELECT
        pair,
        argMaxMerge(last_price_state) AS last,
        argMinMerge(open_price_state) AS open,
        sumMerge(volume_state) AS volume
    FROM agg_crypto_trades_daily
    WHERE event_date = toDate(now(), 'UTC')
    GROUP BY pair
),
quotes AS (
    SELECT
        pair,
        argMaxMerge(bid_state) AS bid,
        argMaxMerge(ask_state) AS ask
    FROM agg_crypto_quotes_daily
    WHERE event_date = toDate(now(), 'UTC')
    GROUP BY pair
)
SELECT
    t.pair,
    t.last AS price,
    t.open,
    ((t.last - t.open) / t.open) * 100 AS change_pct,
    t.volume,
    q.bid,
    q.ask,
    q.ask - q.bid AS spread
FROM trades t
JOIN quotes q ON t.pair = q.pair
ORDER BY t.volume DESC;
```

### 19.4 OHLC with Polymarket Example (CryptoHouse)

```sql
WITH prices AS (
    SELECT
        toStartOfInterval(e.timestamp, INTERVAL 10 second) AS time_interval,
        CASE
            WHEN pos.id IN ('21742633...', '69236923...')
            THEN e.maker_amount_filled / e.taker_amount_filled
            ELSE 1 - (e.maker_amount_filled / e.taker_amount_filled)
        END AS price
    FROM polymarket.orders_filled e
    INNER JOIN polymarket.token_id_condition pos
        ON e.maker_asset_id = pos.id OR e.taker_asset_id = pos.id
    WHERE pos.condition = '0xdd22472e...'
      AND maker_asset_id = '0'
      AND timestamp < '2024-11-05'::Date32
),
filled_prices AS (
    SELECT
        time_interval,
        price,
        IF(price = 0,
           anyLast(price) OVER (ORDER BY time_interval),
           price
        ) AS filled_price
    FROM prices
)
SELECT
    time_interval,
    MIN(filled_price) AS low_price,
    MAX(filled_price) AS high_price,
    any(filled_price) AS open_price,
    anyLast(filled_price) AS close_price
FROM filled_prices
GROUP BY time_interval
ORDER BY time_interval DESC
```

---

# Part 5: Query Patterns

## 20. ARRAY JOIN for Log Parsing

```sql
-- Parse compute units from transaction logs
SELECT
    signature AS id,
    maxIf(
        toInt64OrNull(splitByString(' ', log)[4]),
        log LIKE 'Program % consumed % of % compute units'
    ) AS computed_units_spent
FROM solana.transactions
ARRAY JOIN log_messages AS log
WHERE block_slot = 293312554
GROUP BY id
```

## 21. Balance Change Analysis

```sql
-- Most active account by balance change
SELECT
    account,
    sum(abs(before - after)) AS absolute_balance_change
FROM (
    SELECT
        arrayJoin(balance_changes) AS balance_change,
        balance_change.1 AS account,
        balance_change.2 AS before,
        balance_change.3 AS after
    FROM solana.transactions_non_voting
    WHERE toDate(block_timestamp) = '2024-07-24'
)
GROUP BY account
ORDER BY absolute_balance_change DESC
```

## 22. Compute Unit Analysis with CTEs

```sql
WITH
-- Extract requested compute units from ComputeBudget instruction
computebudget_a AS (
    SELECT
        tx_signature AS id,
        CASE
            WHEN substring(base58Decode(data), 1, 1) = toFixedString('\x02', 1)
            THEN reinterpretAsUInt32(substring(base58Decode(data), 2, 3))
            ELSE 0
        END AS requestedUnits
    FROM solana.instructions
    WHERE block_slot = 293312554
      AND program_id = 'ComputeBudget111111111111111111111111111111'
),
-- Deduplicate (take highest requested)
computebudget AS (
    SELECT id, requestedUnits,
           ROW_NUMBER() OVER (PARTITION BY id ORDER BY requestedUnits DESC) AS rn
    FROM computebudget_a
),
-- Extract consumed units from logs
consumedunits_a AS (
    SELECT
        signature AS id,
        maxIf(
            toInt64OrNull(splitByString(' ', log)[4]),
            log LIKE 'Program % consumed % of % compute units'
        ) AS computed_units_spent
    FROM solana.transactions
    ARRAY JOIN log_messages AS log
    WHERE block_slot = 293312554
    GROUP BY id
),
consumedunits AS (
    SELECT id, MAX(computed_units_spent) AS consumed_units
    FROM consumedunits_a
    GROUP BY id
)
SELECT
    concat('https://explorer.solana.com/tx/', t.signature) AS explorer_link,
    signature,
    CASE WHEN t.err = 'null' THEN '✅' ELSE '❌' END AS success,
    COALESCE(cb.requestedUnits, 0) - COALESCE(cu.consumed_units, 0) AS excessComputeUnit,
    COALESCE(cb.requestedUnits, 0) AS requestedUnits,
    COALESCE(cu.consumed_units, 0) AS consumedUnits
FROM solana.transactions t
LEFT JOIN computebudget cb ON t.signature = cb.id
LEFT JOIN consumedunits cu ON t.signature = cu.id
WHERE block_slot = 293312554 AND cb.rn = 1
ORDER BY excessComputeUnit DESC
```

## 23. Window Functions for Rolling Averages

```sql
SELECT
    toStartOfDay(block_timestamp) AS day,
    count() AS txns,
    ROUND(AVG(txns) OVER (
        ORDER BY day ASC
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    )) AS `7d_avg`
FROM base.transactions
WHERE day > '2023-07-12'
GROUP BY day
ORDER BY day DESC
```

## 24. Cumulative Totals with Window Functions

```sql
SELECT
    toStartOfDay(min_date) AS day,
    count() AS new_users,
    sum(new_users) OVER (ORDER BY day ASC) AS total_addresses
FROM ethereum.first_time_for_address
WHERE day > '2023-07-12'
GROUP BY day
ORDER BY day ASC
```

## 25. WITH FILL for Time Series Gaps

### 25.1 Fill Missing OHLCV Buckets

```sql
-- Ensure no gaps in candlestick data
SELECT
    timestamp,
    symbol,
    open,
    high,
    low,
    close,
    volume
FROM ohlcv_1m
WHERE symbol = 'BTC-USD'
    AND timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp ASC
WITH FILL
    FROM toStartOfMinute(now() - INTERVAL 1 HOUR)
    TO toStartOfMinute(now())
    STEP INTERVAL 1 MINUTE;
```

### 25.2 Fill with Previous Values (Forward Fill)

```sql
SELECT
    timestamp,
    symbol,
    -- Use previous close for missing open
    if(open = 0, lagInFrame(close) OVER (ORDER BY timestamp), open) AS open,
    if(high = 0, lagInFrame(close) OVER (ORDER BY timestamp), high) AS high,
    if(low = 0, lagInFrame(close) OVER (ORDER BY timestamp), low) AS low,
    if(close = 0, lagInFrame(close) OVER (ORDER BY timestamp), close) AS close,
    volume
FROM (
    SELECT *
    FROM ohlcv_1m
    WHERE symbol = 'BTC-USD'
    ORDER BY timestamp
    WITH FILL
        FROM toStartOfMinute(now() - INTERVAL 1 HOUR)
        TO toStartOfMinute(now())
        STEP INTERVAL 1 MINUTE
)
```

### 25.3 Fill Grid for Heatmaps

```sql
-- Complete hour x day grid
SELECT
    hour,
    day_of_week,
    count
FROM trades_by_hour_dow
ORDER BY hour, day_of_week
WITH FILL
    FROM 0 TO 24         -- Hours 0-23
    STEP 1
WITH FILL
    FROM 1 TO 8          -- Days 1-7
    STEP 1;
```

## 26. Conditional Aggregates (maxIf, sumIf)

```sql
-- maxIf: Aggregate only when condition is true
SELECT
    signature AS id,
    maxIf(
        toInt64OrNull(splitByString(' ', log)[4]),
        log LIKE 'Program % consumed % of % compute units'
    ) AS computed_units_spent
FROM solana.transactions
ARRAY JOIN log_messages AS log
WHERE block_slot = 293312554
GROUP BY id

-- sumIf example for trading
SELECT
    toDate(block_timestamp) AS date,
    token_symbol,
    sumIf(volume_usd, trade_direction = 0) AS buy_volume,
    sumIf(volume_usd, trade_direction = 1) AS sell_volume,
    countIf(trade_direction = 0) AS buy_count,
    countIf(trade_direction = 1) AS sell_count
FROM enriched_trades
GROUP BY date, token_symbol
```

## 27. Safe Type Casting

```sql
-- toInt64OrNull: Returns NULL instead of error on invalid input
toInt64OrNull(splitByString(' ', log)[4])

-- ::Type casting syntax (PostgreSQL-style)
JSONExtractFloat(raw_data, 'price')::Decimal(38, 18)
'2024-11-05'::Date32
maker_asset_id::text
block_timestamp::Date AS date

-- parseDateTimeBestEffort: Flexible timestamp parsing
parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp'))
```

## 28. Parameterized Queries

```sql
-- ClickHouse parameterized query syntax
SELECT uniqCombinedMerge(14)(trx_count) AS unique_transactions
FROM solana.tx_signature_by_day_per_program
WHERE program_id = {program_id:String}    -- Parameter with type

-- Common parameter types:
-- {param:String}
-- {param:UInt64}
-- {param:Date}
-- {param:DateTime}

-- Usage in API:
-- POST /query?program_id=TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
```

## 29. Interval Functions

```sql
-- toIntervalWeek, toIntervalDay, toIntervalMonth
WHERE day >= (today() - toIntervalWeek(1))
WHERE day > (today() - toIntervalMonth(6))
WHERE t >= toUnixTimestamp64Milli(now64() - INTERVAL 10 MINUTE)

-- now() - INTERVAL syntax
now() - INTERVAL 1 WEEK
now() - INTERVAL 1 HOUR
now64() - INTERVAL 30 MINUTE
```

## 30. any() and anyLast() Patterns

```sql
-- any(): First value encountered (for OPEN price)
-- anyLast(): Last value encountered (for CLOSE price)
SELECT
    time_interval,
    any(filled_price) AS open_price,           -- First value
    anyLast(filled_price) AS close_price,      -- Last value
    MIN(filled_price) AS low_price,
    MAX(filled_price) AS high_price
FROM filled_prices
GROUP BY time_interval

-- anyLast() OVER for forward fill
SELECT
    time_interval,
    price,
    IF(price = 0,
       anyLast(price) OVER (ORDER BY time_interval),
       price
    ) AS filled_price
FROM prices
```

## 31. numbers() Table Function

```sql
-- Generate sequence for testing or filling
SELECT count() FROM numbers(10000000000)  -- 10 billion rows

-- Generate date range
SELECT toDate('2024-01-01') + number AS date
FROM numbers(365)

-- Generate time series skeleton
SELECT
    toDateTime('2024-01-01 00:00:00') + INTERVAL number MINUTE AS timestamp
FROM numbers(60 * 24)  -- 1440 minutes in a day
```

---

# Part 6: JSON Handling Patterns

## 32. JSON SKIP Clause (Skip Large Nested Fields)

```sql
-- Skip large/unused nested fields to save storage
CREATE TABLE orders_raw (
    data JSON(
        SKIP `nested.large_field`,           -- Skip specific paths
        SKIP `audit.full_history`,
        SKIP `metadata.raw_response`
    ),
    order_id String MATERIALIZED JSONExtractString(data, 'order_id'),
    symbol LowCardinality(String) MATERIALIZED JSONExtractString(data, 'symbol'),
    timestamp DateTime64(3) MATERIALIZED parseDateTimeBestEffort(JSONExtractString(data, 'timestamp'))
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, order_id);
```

## 33. isValidJSON() for Filtering

```sql
-- Filter invalid JSON before processing
CREATE MATERIALIZED VIEW trades_mv TO trades AS
SELECT *
FROM trades_queue
WHERE isValidJSON(data) = 1;

-- Query to find invalid records
SELECT count() AS invalid_count
FROM trades_queue
WHERE isValidJSON(data) = 0;
```

## 34. MATERIALIZED Columns from JSON

```sql
CREATE TABLE events (
    raw_json String,
    -- Computed once at insert time, not query time
    event_type LowCardinality(String) MATERIALIZED JSONExtractString(raw_json, 'type'),
    timestamp DateTime64(3) MATERIALIZED parseDateTimeBestEffort(JSONExtractString(raw_json, 'ts')),
    symbol LowCardinality(String) MATERIALIZED JSONExtractString(raw_json, 'symbol'),
    dedup_hash UInt64 MATERIALIZED cityHash64(raw_json)
)
ENGINE = ReplacingMergeTree
ORDER BY (event_type, timestamp, dedup_hash);
```

---

# Part 7: Deduplication and Data Fixes

## 35. Deduplication with cityHash64

### 35.1 Hash-Based Deduplication

```sql
-- Create dedup hash from key fields
CREATE TABLE trades (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    trade_id String,
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    -- Computed dedup hash
    dedup_hash UInt64 MATERIALIZED cityHash64(
        symbol,
        exchange,
        trade_id,
        toString(timestamp)
    )
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, dedup_hash);
```

### 35.2 Query with Deduplication

```sql
-- Use FINAL to get deduplicated results
SELECT *
FROM trades FINAL
WHERE symbol = 'BTC-USD'
    AND timestamp >= now() - INTERVAL 1 HOUR;

-- Or use subquery for better performance on large tables
SELECT *
FROM (
    SELECT *,
        row_number() OVER (PARTITION BY dedup_hash ORDER BY timestamp DESC) AS rn
    FROM trades
    WHERE symbol = 'BTC-USD'
)
WHERE rn = 1;
```

## 36. Lightweight Deletes for Data Fixes

### 36.1 Enable Lightweight Delete Mode

```sql
-- Enable lightweight deletes (faster than traditional mutations)
SET lightweight_delete_mode = 'lightweight_update';
```

### 36.2 Delete Specific Date from Aggregate Tables

```sql
-- Delete bad data from all aggregate tables
ALTER TABLE trades_per_day DELETE
WHERE date = CAST('2024-01-15', 'Date');

ALTER TABLE trades_per_day_by_symbol DELETE
WHERE date = CAST('2024-01-15', 'Date');

ALTER TABLE trades_per_day_by_symbol_by_exchange DELETE
WHERE date = CAST('2024-01-15', 'Date');

-- Delete from main table
ALTER TABLE trades DELETE
WHERE toDate(timestamp) = CAST('2024-01-15', 'Date');
```

## 37. Full Rebuild Pattern (Drop MV, Rebuild, Exchange)

```sql
-- 1. Drop the materialized view
DROP VIEW IF EXISTS trades_per_month_mv;

-- 2. Create shadow table
CREATE TABLE trades_per_month_v2 AS trades_per_month;

-- 3. Rebuild from source data
INSERT INTO trades_per_month_v2
SELECT
    toStartOfMonth(date) AS month,
    symbol,
    sum(count) AS count,
    sum(volume) AS volume
FROM trades_per_day
GROUP BY month, symbol;

-- 4. Atomic swap
EXCHANGE TABLES trades_per_month_v2 AND trades_per_month;

-- 5. Cleanup
DROP TABLE trades_per_month_v2;

-- 6. Recreate MV
CREATE MATERIALIZED VIEW trades_per_month_mv
TO trades_per_month AS
SELECT
    toStartOfMonth(timestamp) AS month,
    symbol,
    count() AS count,
    sum(quantity) AS volume
FROM trades
GROUP BY month, symbol;
```

## 38. Atomic Table Swap (EXCHANGE TABLES)

```sql
-- 1. Create temp table with same schema
CREATE TABLE IF NOT EXISTS default.queries_temp
(
    id String DEFAULT generateULID(),
    name String,
    group String,
    query String,
    chart String DEFAULT '{"type":"line"}',
    format Bool,
    params String DEFAULT '{}'
)
ENGINE = MergeTree()
ORDER BY id;

-- 2. Load data into temp table
INSERT INTO default.queries_temp SELECT * FROM ...;

-- 3. Atomic swap (instant, no downtime)
EXCHANGE TABLES default.queries_temp AND default.queries;

-- 4. Cleanup
DROP TABLE default.queries_temp;
```

**Key Benefit**: Updates are atomic - users always see either old or new data, never partial state.

---

# Part 8: Security Model

## 39. Read-Only User with Limits

```sql
CREATE USER crypto
IDENTIFIED WITH double_sha1_password BY 'hash...'
DEFAULT ROLE NONE
SETTINGS
    readonly = 1,
    add_http_cors_header = true,
    max_execution_time = 60,
    max_rows_to_read = 10000000000,
    max_bytes_to_read = 1000000000000,
    max_network_bandwidth = 25000000,
    max_memory_usage = 20000000000,
    max_bytes_before_external_group_by = 10000000000,
    max_result_rows = 10000,
    max_result_bytes = 10000000,
    result_overflow_mode = 'throw',
    read_overflow_mode = 'throw'
```

## 40. Settings Profile with Query Cache

```sql
CREATE SETTINGS PROFILE crypto
SETTINGS
    readonly = 1,
    add_http_cors_header = true,
    max_execution_time = 60,
    max_rows_to_read = 10000000000,
    max_bytes_to_read = 1000000000000,
    max_network_bandwidth = 25000000,
    max_memory_usage = 20000000000,
    max_bytes_before_external_group_by = 10000000000,
    max_result_rows = 1000,
    max_result_bytes = 10000000,
    result_overflow_mode = 'break' CHANGEABLE_IN_READONLY,
    read_overflow_mode = 'break' CHANGEABLE_IN_READONLY,
    enable_http_compression = true,
    use_query_cache = true,                              -- Enable query caching
    query_cache_nondeterministic_function_handling = 'save' CHANGEABLE_IN_READONLY
TO crypto
```

## 41. Granular Permissions

```sql
-- Database-level grants
GRANT SELECT ON solana.* TO crypto;
GRANT SELECT ON ethereum.* TO crypto;
GRANT SELECT ON polymarket.* TO crypto;

-- Table-level grants
GRANT SELECT ON default.queries TO crypto;

-- Column-level grants (monitor user)
GRANT SELECT(elapsed, initial_user, query_id, read_bytes, read_rows, total_rows_approx)
ON system.processes TO monitor;

-- Special permissions
GRANT KILL QUERY ON *.* TO monitor_admin;
```

## 42. IP-Based Quotas

```sql
CREATE QUOTA crypto
KEYED BY ip_address
FOR INTERVAL 1 hour MAX
    queries = 60,
    result_rows = 3000000000,
    read_rows = 3000000000000,
    execution_time = 6000
TO crypto;

CREATE QUOTA monitor
KEYED BY ip_address
FOR INTERVAL 1 hour MAX queries = 1200
TO monitor;
```

---

# Part 9: TTL and Performance Settings

## 43. ttl_only_drop_parts (Faster TTL Cleanup)

```sql
-- Drop entire parts instead of row-by-row deletion
CREATE TABLE tick_data (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (symbol, timestamp)
TTL timestamp + INTERVAL 7 DAY
SETTINGS ttl_only_drop_parts = 1;      -- Key setting!
```

**How it works:**
- Instead of row-by-row deletion (slow, creates rewrites)
- Drops entire parts when all rows exceed TTL
- Requires partition alignment with TTL period
- 100x faster TTL cleanup

## 44. force_primary_key (Prevent Full Scans)

```sql
-- Require queries to use primary key
SET force_primary_key = 1;

-- This will fail (no primary key filter):
SELECT * FROM trades WHERE price > 100;  -- Error!

-- This will work:
SELECT * FROM trades WHERE symbol = 'BTC-USD' AND price > 100;  -- OK
```

## 45. do_not_merge_across_partitions_select_final

```sql
-- Optimize FINAL queries on partitioned tables
SET do_not_merge_across_partitions_select_final = 1;

-- Now FINAL only deduplicates within each partition (much faster)
SELECT *
FROM trades FINAL
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-02-01';
```

## 46. Query Cache Settings

```sql
-- Enable query caching
SET use_query_cache = 1;
SET query_cache_nondeterministic_function_handling = 'save';

-- Repeated queries return cached results
SELECT * FROM v_trending_tokens LIMIT 100;  -- Cached after first call
```

## 47. Memory and Execution Limits

```sql
-- For dashboard queries
SET max_execution_time = 30;              -- 30 second timeout
SET max_rows_to_read = 100000000;         -- 100M row limit
SET max_bytes_to_read = 10000000000;      -- 10GB limit

-- For analytics queries
SET max_memory_usage = 20000000000;       -- 20GB memory
SET max_bytes_before_external_group_by = 10000000000;  -- Spill to disk
```

## 48. Table-Level Settings

```sql
CREATE TABLE high_frequency_data (...)
ENGINE = MergeTree
...
SETTINGS
    ttl_only_drop_parts = 1,              -- Fast TTL cleanup
    index_granularity = 8192,             -- Default, good for most cases
    merge_with_ttl_timeout = 86400,       -- TTL merge every 24h
    min_bytes_for_wide_part = 10485760    -- 10MB threshold for wide parts
```

---

# Part 10: Functions Reference

## 49. Crypto-Specific Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `base58Decode(s)` | Decode Solana base58 data | `base58Decode(instruction_data)` |
| `generateULID()` | Sortable unique ID | `DEFAULT generateULID()` |
| `reinterpretAsUInt32(s)` | Binary to integer | Parse instruction bytes |
| `splitByString(sep, s)` | Parse log messages | `splitByString(' ', log)[4]` |
| `cityHash64(...)` | Fast hash for dedup | `cityHash64(symbol, trade_id)` |

## 50. Time Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `toStartOfDay(ts)` | Daily aggregation | `toStartOfDay(block_timestamp)` |
| `toStartOfMinute(ts)` | Minute OHLCV | `toStartOfMinute(timestamp)` |
| `toStartOfInterval(ts, INTERVAL)` | Custom intervals | `toStartOfInterval(ts, INTERVAL 10 second)` |
| `toStartOfWeek(ts)` | Weekly aggregation | `toStartOfWeek(timestamp)` |
| `toStartOfMonth(ts)` | Monthly aggregation | `toStartOfMonth(timestamp)` |
| `toYYYYMMDD(ts)` | Date partition | `PARTITION BY toYYYYMMDD(ts)` |
| `toYYYYMM(ts)` | Month partition | `PARTITION BY toYYYYMM(ts)` |
| `parseDateTimeBestEffort(s)` | Flexible parsing | Parse various timestamp formats |
| `toUnixTimestamp64Milli(ts)` | DateTime to epoch ms | Store as UInt64 |
| `fromUnixTimestamp64Milli(ms, tz)` | Epoch ms to DateTime | Convert from UInt64 |

## 51. Array Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `arrayExists(lambda, arr)` | Check condition | Filter vote transactions |
| `arrayJoin(arr)` | Unnest array | Expand balance_changes |
| `arr[n].field` | Access tuple field | `accounts[1].1` (first account pubkey) |
| `arrayFirst(lambda, arr)` | Find first match | Get first matching element |
| `arrayFilter(lambda, arr)` | Filter array | Remove unwanted elements |

## 52. Aggregate State Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `uniqCombinedState(14)(col)` | HyperLogLog state | Distinct count storage |
| `uniqCombinedMerge(14)(col)` | Merge HLL states | Query aggregate table |
| `sumState(col)` | Sum state | Store for incremental |
| `sumMerge(col)` | Merge sum states | Query aggregate table |
| `avgState(col)` | Average state | Store running average |
| `avgMerge(col)` | Merge avg states | Query aggregate table |
| `argMinState(val, ts)` | Value at min ts | OPEN price |
| `argMaxState(val, ts)` | Value at max ts | CLOSE price |
| `argMinMerge(state)` | Merge argMin | Query OPEN |
| `argMaxMerge(state)` | Merge argMax | Query CLOSE |

---

# Part 11: Output Formats

## 53. Arrow Format (Efficient Binary)

```javascript
// Efficient binary format for large datasets
const response = await fetch(url, {
    method: 'POST',
    body: query + ' FORMAT Arrow'
});
```

## 54. JSONCompact Format (API Responses)

```javascript
// Human-readable, smaller payload for APIs
const response = await fetch(url, {
    method: 'POST',
    body: query + ' FORMAT JSONCompact'
});
```

---

# Part 12: Summary Tables

## 55. All Patterns Summary

| Pattern | Source | Click Relevance |
|---------|--------|-----------------|
| **ReplacingMergeTree everywhere** | CryptoHouse | Handle chain reorgs naturally |
| **DateTime64(6)** | CryptoHouse | Microsecond timestamps |
| **Decimal(38,9) / (76,38)** | CryptoHouse | Exact financial math |
| **Array(Tuple(...))** | CryptoHouse | Nested blockchain data |
| **Null engine + MV** | CryptoHouse | String → typed transformation |
| **arrayExists with lambda** | CryptoHouse | Filter vote transactions |
| **AggregatingMergeTree** | CryptoHouse | Pre-computed metrics |
| **SummingMergeTree** | CryptoHouse | Simple counters |
| **generateULID()** | CryptoHouse | Sortable unique IDs |
| **IP-based quotas** | CryptoHouse | Public API protection |
| **Query cache** | CryptoHouse | Faster dashboard loads |
| **EXCHANGE TABLES** | CryptoHouse | Atomic table swap |
| **Time-bucketed ORDER BY** | StockHouse | High-frequency trade ordering |
| **argMinState/argMaxState** | StockHouse | OHLC open/close prices |
| **AggregateFunction for OHLC** | StockHouse | Daily aggregations |
| **Monthly PARTITION BY** | StockHouse | Time-series partitioning |
| **toUnixTimestamp64Milli** | StockHouse | Millisecond timestamps |
| **fromUnixTimestamp64Milli** | StockHouse | Epoch to DateTime |
| **toStartOfInterval** | StockHouse | Candlestick bucketing |
| **LZ4 compression** | StockHouse | Fast ingestion |
| **50k batch size** | StockHouse | Optimal throughput |
| **Backpressure/load shedding** | StockHouse | Handle bursts |
| **Arrow format** | StockHouse | Efficient data transfer |
| **Enum8** | ClickPy | Order status, event types |
| **lightweight_delete_mode** | ClickPy | Fast data fixes |
| **DROP + rebuild + EXCHANGE** | ClickPy | Atomic aggregate rebuild |
| **Multi-dim SummingMergeTree** | ClickPy | Query-specific aggregates |
| **S3Queue engine** | sql.clickhouse.com | Ingest from S3/GCS |
| **Medallion architecture** | Bluesky | Bronze → Silver → Gold |
| **DLQ pattern** | Bluesky | Handle bad trades/quotes |
| **isValidJSON()** | Bluesky | Filter malformed data |
| **JSON SKIP clause** | Bluesky | Skip large nested fields |
| **MATERIALIZED columns** | Bluesky | Compute at insert time |
| **cityHash64 dedup** | Bluesky | Hash-based deduplication |
| **WITH FILL** | reversedns.space | Fill OHLCV time gaps |
| **force_primary_key** | reversedns.space | Prevent full scans |
| **ttl_only_drop_parts** | Bluesky | Faster TTL cleanup |
| **do_not_merge_across_partitions_select_final** | Bluesky | Faster FINAL queries |

## 56. Optimization Checklist

### Schema Design
- [ ] Use `Decimal(38, 18)` for all financial amounts
- [ ] Use `DateTime64(9)` for nanosecond timestamps
- [ ] Use `LowCardinality(String)` for categorical columns
- [ ] Consider `Enum8` for fixed categories (< 256 values)
- [ ] Apply appropriate codecs (DoubleDelta, Delta, T64, ZSTD)
- [ ] Use `Array(Tuple(...))` for nested data
- [ ] Use `generateULID()` for sortable unique IDs

### Table Engines
- [ ] `ReplacingMergeTree` for state tables (positions, metrics)
- [ ] `SummingMergeTree` for simple counters
- [ ] `AggregatingMergeTree` for unique counts and complex aggregates
- [ ] `Null` engine for staging tables

### ORDER BY
- [ ] Most filtered column first
- [ ] Consider time-bucketed ORDER BY for high-frequency data
- [ ] Include unique identifier as last column
- [ ] Low cardinality columns before high cardinality

### Partitioning & TTL
- [ ] Align partition granularity with TTL period
- [ ] Enable `ttl_only_drop_parts = 1`
- [ ] Daily partitions for high-frequency, monthly for low-frequency

### Queries
- [ ] Use `FINAL` or handle dedup in application
- [ ] Enable `force_primary_key` to catch full scans
- [ ] Use query cache for repeated queries
- [ ] Use `argMin/argMax` for OHLC open/close
- [ ] Use `uniqCombined(14)` for unique counts

### Ingestion
- [ ] Use Null engine staging + MV for transformation
- [ ] NATS/Kafka queues on single node only
- [ ] Batch inserts (50K rows optimal)
- [ ] LZ4 compression for speed
- [ ] Implement DLQ for bad records

### Security
- [ ] Create read-only users with limits
- [ ] Implement IP-based quotas
- [ ] Use settings profiles for different user types

---

## References

- [CryptoHouse](https://github.com/ClickHouse/CryptoHouse) - Solana blockchain analytics (70+ patterns)
- [StockHouse](https://github.com/ClickHouse/stockhouse) - Real-time market data
- [sql.clickhouse.com](https://sql.clickhouse.com) - S3Queue and pipeline patterns
- [ClickHouse Documentation](https://clickhouse.com/docs)
- [Click DataStreams Architecture](./BLOG_01_ARCHITECTURE_OVERVIEW.md)
