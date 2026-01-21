# ClickHouse Implementation Guide for Click

## Industry Context

This guide applies proven patterns from crypto/fintech companies using ClickHouse at scale:

| Company | Use Case | Scale |
|---------|----------|-------|
| **Coinhall** | Multi-chain blockchain analytics | Billions of transactions |
| **Longbridge** | Real-time trading platform | Millions of trades/second |
| **Binance** | Exchange analytics | Petabytes of market data |
| **Cloudflare** | Real-time analytics | 25M+ events/second |

Click's requirements align closely with these patterns: high-throughput blockchain event ingestion, real-time market data queries, and analytical aggregations.

---

## 1. Architecture Overview

### Data Flow
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLICKHOUSE DATA PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                   │
│  │Solana Engine │    │  DataStreams │    │HL Indexer    │                   │
│  │(Raw Events)  │    │(Derived Data)│    │(Perps Data)  │                   │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                   │                            │
│         ▼                   ▼                   ▼                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                           NATS                                       │    │
│  │                    (Message Bus)                                     │    │
│  └─────────────────────────────┬───────────────────────────────────────┘    │
│                                │                                             │
│                                ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    CLICKHOUSE INGESTER                               │    │
│  │                                                                      │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │    │
│  │  │ Consumer 1 │  │ Consumer 2 │  │ Consumer 3 │  │ Consumer N │    │    │
│  │  │ (trades)   │  │ (tokens)   │  │ (OHLCV)    │  │ (HL fills) │    │    │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘    │    │
│  │        │               │               │               │            │    │
│  │        ▼               ▼               ▼               ▼            │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │              BATCH BUFFER (10K-50K rows)                     │   │    │
│  │  └─────────────────────────────┬───────────────────────────────┘   │    │
│  │                                │                                    │    │
│  └────────────────────────────────┼────────────────────────────────────┘    │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         CLICKHOUSE CLUSTER                           │    │
│  │                                                                      │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │                      HOT TIER (NVMe)                         │   │    │
│  │  │                    Recent 7-30 days                          │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  │                                │                                    │    │
│  │                                ▼ (TTL migration)                    │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │                      COLD TIER (S3)                          │   │    │
│  │  │                    Historical data                           │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          REST API                                    │    │
│  │                   (Query Layer)                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Ingestion Format** | RowBinary | 2-3x faster than JSON |
| **Batching** | 10K-50K rows | Balance latency vs throughput |
| **Deduplication** | ReplacingMergeTree | Handle blockchain replay/duplicates |
| **Partitioning** | Daily (trades), Monthly (tokens) | Optimize time-range queries |
| **Storage Tiering** | NVMe → S3 | Cost optimization (>50% savings) |

---

## 2. Table Schemas

### 2.1 Core Tables (Priority 0)

#### trades_nats_v3 (Raw Trades)
```sql
CREATE TABLE trades_nats_v3
(
    -- Primary identifiers
    tx_signature String,
    slot UInt64,

    -- Token info
    token_address String,
    pool_address String,

    -- Trade details
    trade_type Enum8('buy' = 1, 'sell' = 2),
    token_amount Decimal128(18),
    sol_amount Decimal128(9),
    trade_total_usd Decimal64(6),
    price_usd Decimal64(18),

    -- Trader info
    trader_wallet String,

    -- Metadata
    dex_source LowCardinality(String),
    tx_timestamp DateTime64(3),

    -- Ingestion tracking
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(tx_timestamp)
ORDER BY (token_address, tx_timestamp, tx_signature)
TTL tx_timestamp + INTERVAL 30 DAY TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

**Why ReplacingMergeTree?**
- Blockchain indexers may replay events
- NATS at-least-once delivery can cause duplicates
- `tx_signature` provides natural deduplication key

#### ohlcv_candles (Chart Data)
```sql
CREATE TABLE ohlcv_candles
(
    -- Identifiers
    token_address String,
    timeframe Enum8(
        '1s' = 1, '5s' = 2, '15s' = 3, '30s' = 4,
        '1m' = 5, '5m' = 6, '15m' = 7, '30m' = 8,
        '1h' = 9, '4h' = 10, '6h' = 11, '12h' = 12,
        '1d' = 13, '1w' = 14
    ),
    candle_timestamp DateTime64(3),

    -- OHLCV data
    open Decimal64(18),
    high Decimal64(18),
    low Decimal64(18),
    close Decimal64(18),
    volume_usd Decimal64(6),
    volume_token Decimal128(18),
    trade_count UInt32,

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY (timeframe, toYYYYMM(candle_timestamp))
ORDER BY (token_address, timeframe, candle_timestamp)
TTL candle_timestamp + INTERVAL 1 YEAR TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

**Partitioning Strategy:**
- Partition by timeframe AND month
- Queries typically filter by timeframe first
- Prevents scanning unnecessary timeframe data

#### enriched_trades (Trades with Wallet Badges)
```sql
CREATE TABLE enriched_trades
(
    -- Primary identifiers
    tx_signature String,

    -- Token info
    token_address String,

    -- Trade details
    trade_type Enum8('buy' = 1, 'sell' = 2),
    token_amount Decimal128(18),
    trade_total_usd Decimal64(6),
    trade_mcap_usd Decimal64(6),
    price_usd Decimal64(18),

    -- Trader info
    trader_wallet String,

    -- Wallet classifications (from DataStreams)
    is_sniper UInt8,
    is_insider UInt8,
    is_bundler UInt8,
    is_dev UInt8,
    is_whale UInt8,
    is_kol UInt8,
    is_fresh_wallet UInt8,
    is_top_10 UInt8,

    -- Timestamps
    tx_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(tx_timestamp)
ORDER BY (token_address, tx_timestamp, tx_signature)
TTL tx_timestamp + INTERVAL 90 DAY TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

#### tokens_nats_v2 (Token Metadata)
```sql
CREATE TABLE tokens_nats_v2
(
    -- Primary identifier
    token_address String,

    -- Metadata
    token_name String,
    token_symbol String,
    token_image_url String,
    decimals UInt8,
    total_supply Decimal128(18),

    -- Creator info
    creator_wallet String,
    deployment_slot UInt64,
    deployment_timestamp DateTime64(3),

    -- Authorities
    mint_authority Nullable(String),
    freeze_authority Nullable(String),

    -- Social links
    website_url String DEFAULT '',
    twitter_url String DEFAULT '',
    telegram_url String DEFAULT '',
    discord_url String DEFAULT '',

    -- Classification
    launchpad LowCardinality(String),
    dex_source LowCardinality(String),

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYYYYMM(deployment_timestamp)
ORDER BY (token_address)
SETTINGS index_granularity = 8192;
```

#### transfers_nats_v2 (Token Transfers)
```sql
CREATE TABLE transfers_nats_v2
(
    -- Primary identifiers
    tx_signature String,

    -- Token info
    token_address String,

    -- Transfer details
    from_wallet String,
    to_wallet String,
    amount Decimal128(18),

    -- Timestamps
    tx_timestamp DateTime64(3),
    slot UInt64,

    -- Ingestion tracking
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(tx_timestamp)
ORDER BY (token_address, tx_timestamp, tx_signature)
TTL tx_timestamp + INTERVAL 90 DAY TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

#### pools_nats_v1_0_0 (Liquidity Pools)
```sql
CREATE TABLE pools_nats_v1_0_0
(
    -- Primary identifier
    pool_address String,

    -- Token info
    token_address String,
    base_token LowCardinality(String), -- SOL, USDC

    -- Reserves
    reserve_token Decimal128(18),
    reserve_base Decimal128(18),
    liquidity_usd Decimal64(6),

    -- LP tokens
    lp_supply Decimal128(18),
    lp_burned Decimal128(18),
    burned_pct Decimal32(4),

    -- Metadata
    amm_type LowCardinality(String),
    creation_timestamp DateTime64(3),
    creation_slot UInt64,

    -- Status
    is_graduated UInt8,
    graduation_timestamp Nullable(DateTime64(3)),

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYYYYMM(creation_timestamp)
ORDER BY (token_address, pool_address)
SETTINGS index_granularity = 8192;
```

#### token_metrics_nats_v2 (Real-time Metrics Snapshots)
```sql
CREATE TABLE token_metrics_nats_v2
(
    -- Primary identifier
    token_address String,
    snapshot_timestamp DateTime64(3),

    -- Price metrics
    price_usd Decimal64(18),
    price_sol Decimal64(18),
    market_cap_usd Decimal64(6),
    ath_market_cap_usd Decimal64(6),

    -- Price changes
    price_change_5m_pct Decimal32(4),
    price_change_1h_pct Decimal32(4),
    price_change_6h_pct Decimal32(4),
    price_change_24h_pct Decimal32(4),

    -- Volume metrics
    volume_24h_usd Decimal64(6),
    volume_1h_usd Decimal64(6),
    buy_volume_24h_usd Decimal64(6),
    sell_volume_24h_usd Decimal64(6),

    -- Holder metrics
    holder_count UInt32,
    top_10_holder_pct Decimal32(4),
    top_20_holder_pct Decimal32(4),

    -- Risk metrics
    dev_holdings_pct Decimal32(4),
    sniper_pct Decimal32(4),
    insider_pct Decimal32(4),
    bundler_pct Decimal32(4),
    rug_risk_pct Decimal32(4),

    -- Liquidity
    liquidity_usd Decimal64(6),
    burned_lp_pct Decimal32(4),

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYYYYMMDD(snapshot_timestamp)
ORDER BY (token_address, snapshot_timestamp)
TTL snapshot_timestamp + INTERVAL 7 DAY
SETTINGS index_granularity = 8192;
```

### 2.2 User Data Tables (Priority 0)

#### position_overview (User Positions)
```sql
CREATE TABLE position_overview
(
    -- User identification
    user_id String,
    wallet_address String,

    -- Token info
    token_address String,

    -- Position metrics
    bought_sol Decimal64(9),
    bought_usd Decimal64(6),
    sold_sol Decimal64(9),
    sold_usd Decimal64(6),
    current_balance Decimal128(18),
    current_value_usd Decimal64(6),

    -- PnL
    realized_pnl_sol Decimal64(9),
    realized_pnl_usd Decimal64(6),
    unrealized_pnl_sol Decimal64(9),
    unrealized_pnl_usd Decimal64(6),
    total_pnl_pct Decimal32(4),

    -- Timestamps
    first_buy_timestamp DateTime64(3),
    last_activity_timestamp DateTime64(3),
    snapshot_timestamp DateTime64(3),

    -- Status
    is_hidden UInt8 DEFAULT 0,

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYYYYMM(snapshot_timestamp)
ORDER BY (user_id, token_address, snapshot_timestamp)
TTL snapshot_timestamp + INTERVAL 1 YEAR TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

#### order_overview (Order Summaries)
```sql
CREATE TABLE order_overview
(
    -- Order identification
    order_id String,
    user_id String,

    -- Token info
    token_address String,

    -- Order details
    order_type Enum8('market_buy' = 1, 'market_sell' = 2, 'limit_buy' = 3, 'limit_sell' = 4, 'take_profit' = 5, 'stop_loss' = 6),
    order_status Enum8('pending' = 1, 'placed' = 2, 'partially_filled' = 3, 'filled' = 4, 'cancelled' = 5, 'failed' = 6),

    -- Amounts
    requested_amount_sol Decimal64(9),
    filled_amount_sol Decimal64(9),
    filled_amount_usd Decimal64(6),

    -- Trigger conditions (for limit orders)
    trigger_mcap_usd Nullable(Decimal64(6)),
    trigger_price_usd Nullable(Decimal64(18)),

    -- Execution
    tx_signature Nullable(String),
    executed_price_usd Nullable(Decimal64(18)),
    executed_mcap_usd Nullable(Decimal64(6)),

    -- Timestamps
    created_at DateTime64(3),
    updated_at DateTime64(3),
    executed_at Nullable(DateTime64(3)),
    expires_at Nullable(DateTime64(3)),

    -- Ingestion tracking
    _version UInt64 DEFAULT toUnixTimestamp64Milli(now64(3))
)
ENGINE = ReplacingMergeTree(_version)
PARTITION BY toYYYYMM(created_at)
ORDER BY (user_id, created_at, order_id)
TTL created_at + INTERVAL 1 YEAR TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

### 2.3 Hyperliquid Tables (Priority 1)

#### hl_fills (Perps Trades)
```sql
CREATE TABLE hl_fills
(
    -- Primary identifier
    fill_id String,

    -- User info
    user_address String,

    -- Trade details
    coin LowCardinality(String),
    direction Enum8('long' = 1, 'short' = 2),
    size Decimal64(8),
    price Decimal64(8),
    notional_value Decimal64(6),

    -- Fees and PnL
    fee Decimal64(8),
    realized_pnl Decimal64(8),

    -- Order info
    order_id String,
    is_maker UInt8,

    -- Timestamps
    fill_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(fill_timestamp)
ORDER BY (user_address, fill_timestamp, fill_id)
TTL fill_timestamp + INTERVAL 1 YEAR TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

#### hl_funding (Funding Payments)
```sql
CREATE TABLE hl_funding
(
    -- Primary identifier
    funding_id String,

    -- User info
    user_address String,

    -- Funding details
    coin LowCardinality(String),
    funding_rate Decimal64(10),
    funding_payment Decimal64(8),
    position_size Decimal64(8),

    -- Timestamps
    funding_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(funding_timestamp)
ORDER BY (user_address, funding_timestamp, funding_id)
TTL funding_timestamp + INTERVAL 1 YEAR TO VOLUME 'cold'
SETTINGS index_granularity = 8192;
```

#### hl_liquidations (Liquidation Events)
```sql
CREATE TABLE hl_liquidations
(
    -- Primary identifier
    liquidation_id String,

    -- User info
    user_address String,

    -- Liquidation details
    coin LowCardinality(String),
    direction Enum8('long' = 1, 'short' = 2),
    size Decimal64(8),
    price Decimal64(8),
    liquidation_value Decimal64(6),

    -- Timestamps
    liquidation_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(liquidation_timestamp)
ORDER BY (user_address, liquidation_timestamp, liquidation_id)
SETTINGS index_granularity = 8192;
```

#### hl_deposits / hl_withdrawals
```sql
CREATE TABLE hl_deposits
(
    deposit_id String,
    user_address String,
    amount Decimal64(8),
    coin LowCardinality(String),
    tx_hash String,
    deposit_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(deposit_timestamp)
ORDER BY (user_address, deposit_timestamp, deposit_id)
SETTINGS index_granularity = 8192;

CREATE TABLE hl_withdrawals
(
    withdrawal_id String,
    user_address String,
    amount Decimal64(8),
    coin LowCardinality(String),
    tx_hash String,
    status Enum8('pending' = 1, 'completed' = 2, 'failed' = 3),
    withdrawal_timestamp DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(withdrawal_timestamp)
ORDER BY (user_address, withdrawal_timestamp, withdrawal_id)
SETTINGS index_granularity = 8192;
```

### 2.4 Lifecycle Events (Priority 2)

#### graduation_events
```sql
CREATE TABLE graduation_events
(
    -- Primary identifier
    event_id String,
    tx_signature String,

    -- Token info
    token_address String,
    launchpad LowCardinality(String),

    -- Graduation details
    final_bonding_mcap_usd Decimal64(6),
    initial_pool_liquidity_usd Decimal64(6),
    pool_address String,
    amm_type LowCardinality(String),

    -- Timestamps
    graduation_timestamp DateTime64(3),
    slot UInt64,
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMM(graduation_timestamp)
ORDER BY (token_address, graduation_timestamp)
SETTINGS index_granularity = 8192;
```

#### liquidity_events
```sql
CREATE TABLE liquidity_events
(
    -- Primary identifier
    event_id String,
    tx_signature String,

    -- Pool info
    pool_address String,
    token_address String,

    -- Event details
    event_type Enum8('add' = 1, 'remove' = 2, 'burn' = 3),
    token_amount Decimal128(18),
    base_amount Decimal128(18),
    lp_amount Decimal128(18),
    liquidity_usd Decimal64(6),

    -- Actor
    wallet_address String,

    -- Timestamps
    event_timestamp DateTime64(3),
    slot UInt64,
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(_inserted_at)
PARTITION BY toYYYYMMDD(event_timestamp)
ORDER BY (token_address, event_timestamp, event_id)
SETTINGS index_granularity = 8192;
```

#### failed_transactions
```sql
CREATE TABLE failed_transactions
(
    -- Primary identifier
    tx_signature String,

    -- User info
    user_id String,
    wallet_address String,

    -- Transaction details
    tx_type Enum8('buy' = 1, 'sell' = 2, 'transfer' = 3),
    token_address Nullable(String),
    requested_amount Decimal64(9),

    -- Failure info
    error_code String,
    error_message String,

    -- Timestamps
    attempted_at DateTime64(3),
    _inserted_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(attempted_at)
ORDER BY (user_id, attempted_at, tx_signature)
TTL attempted_at + INTERVAL 30 DAY
SETTINGS index_granularity = 8192;
```

---

## 3. Materialized Views

### 3.1 Volume Aggregations

#### Hourly Volume by Token
```sql
CREATE MATERIALIZED VIEW mv_volume_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (token_address, hour)
AS SELECT
    token_address,
    toStartOfHour(tx_timestamp) AS hour,
    sum(trade_total_usd) AS volume_usd,
    sumIf(trade_total_usd, trade_type = 'buy') AS buy_volume_usd,
    sumIf(trade_total_usd, trade_type = 'sell') AS sell_volume_usd,
    count() AS trade_count,
    countIf(trade_type = 'buy') AS buy_count,
    countIf(trade_type = 'sell') AS sell_count,
    uniq(trader_wallet) AS unique_traders
FROM trades_nats_v3
GROUP BY token_address, hour;
```

#### Daily Volume by Token
```sql
CREATE MATERIALIZED VIEW mv_volume_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (token_address, day)
AS SELECT
    token_address,
    toDate(tx_timestamp) AS day,
    sum(trade_total_usd) AS volume_usd,
    sumIf(trade_total_usd, trade_type = 'buy') AS buy_volume_usd,
    sumIf(trade_total_usd, trade_type = 'sell') AS sell_volume_usd,
    count() AS trade_count,
    uniq(trader_wallet) AS unique_traders,
    max(price_usd) AS high_price,
    min(price_usd) AS low_price
FROM trades_nats_v3
GROUP BY token_address, day;
```

### 3.2 Holder Analytics

#### Current Holder Balances
```sql
CREATE MATERIALIZED VIEW mv_holder_balances
ENGINE = SummingMergeTree()
ORDER BY (token_address, wallet_address)
AS SELECT
    token_address,
    to_wallet AS wallet_address,
    sum(amount) AS balance_change
FROM (
    SELECT token_address, to_wallet, amount FROM transfers_nats_v2
    UNION ALL
    SELECT token_address, from_wallet AS to_wallet, -amount AS amount FROM transfers_nats_v2
)
GROUP BY token_address, wallet_address;
```

### 3.3 User PnL Tracking

#### Realized PnL by User
```sql
CREATE MATERIALIZED VIEW mv_user_realized_pnl
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (user_id, token_address, day)
AS SELECT
    -- Assumes we have user_id mapping via join or in enriched_trades
    trader_wallet AS user_id,
    token_address,
    toDate(tx_timestamp) AS day,
    sumIf(trade_total_usd, trade_type = 'sell') AS total_sold_usd,
    sumIf(trade_total_usd, trade_type = 'buy') AS total_bought_usd,
    count() AS trade_count
FROM trades_nats_v3
GROUP BY trader_wallet, token_address, day;
```

---

## 4. Ingestion Pipeline

### 4.1 NATS Consumer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    NATS CONSUMER POOL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Topic: solana.trades.v3                                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ Consumer 1  │ │ Consumer 2  │ │ Consumer 3  │  (Parallel)   │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘               │
│         └───────────────┼───────────────┘                       │
│                         ▼                                        │
│              ┌─────────────────────┐                            │
│              │   Batch Accumulator  │                            │
│              │   (50K rows / 5s)    │                            │
│              └──────────┬──────────┘                            │
│                         ▼                                        │
│              ┌─────────────────────┐                            │
│              │   RowBinary Encoder  │                            │
│              └──────────┬──────────┘                            │
│                         ▼                                        │
│              ┌─────────────────────┐                            │
│              │   Batch Insert      │                            │
│              │   (ClickHouse)      │                            │
│              └─────────────────────┘                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Ingestion Configuration

```yaml
# Recommended ingester settings
ingester:
  # Batching
  batch_size: 50000           # Rows per batch
  batch_timeout_ms: 5000      # Max wait before flush

  # Parallelism
  consumer_count: 3           # Per topic
  writer_threads: 2           # ClickHouse writers

  # Backpressure
  max_pending_batches: 10     # Buffer limit

  # Retry
  max_retries: 5
  retry_backoff_ms: 1000
  retry_max_backoff_ms: 30000
```

### 4.3 Format Optimization

| Format | Throughput | CPU Usage | Recommendation |
|--------|------------|-----------|----------------|
| JSON | ~30K rows/sec | High | Development only |
| JSONEachRow | ~50K rows/sec | Medium | Simple pipelines |
| **RowBinary** | ~150K rows/sec | Low | **Production** |
| Native | ~200K rows/sec | Lowest | Maximum throughput |

**RowBinary Implementation (Rust):**
```rust
// Pseudo-code for RowBinary encoding
fn encode_trade(trade: &Trade, buffer: &mut Vec<u8>) {
    // String fields: length-prefixed
    write_string(buffer, &trade.tx_signature);
    write_u64(buffer, trade.slot);
    write_string(buffer, &trade.token_address);

    // Enum as u8
    write_u8(buffer, trade.trade_type as u8);

    // Decimal as scaled integer
    write_decimal128(buffer, &trade.token_amount, 18);

    // DateTime64 as milliseconds since epoch
    write_i64(buffer, trade.tx_timestamp.timestamp_millis());
}
```

### 4.4 Async Inserts

For lower-latency requirements:
```sql
-- Enable async inserts for specific tables
ALTER TABLE trades_nats_v3
    MODIFY SETTING async_insert = 1,
                   async_insert_threads = 4,
                   async_insert_max_data_size = 10000000,  -- 10MB
                   async_insert_busy_timeout_ms = 1000;
```

---

## 5. Query Optimization

### 5.1 Common Query Patterns

#### Pattern 1: Token Chart Data
```sql
-- Optimized: Uses partition pruning on timeframe
SELECT
    candle_timestamp,
    open, high, low, close, volume_usd
FROM ohlcv_candles
WHERE token_address = {token:String}
  AND timeframe = {tf:Enum8}
  AND candle_timestamp >= {from:DateTime64}
  AND candle_timestamp <= {to:DateTime64}
ORDER BY candle_timestamp
FORMAT JSONCompact;

-- Expected: < 50ms for 1000 candles
```

#### Pattern 2: Recent Trades
```sql
-- Optimized: ORDER BY matches query, LIMIT prevents full scan
SELECT
    tx_timestamp,
    trade_type,
    token_amount,
    trade_total_usd,
    trader_wallet
FROM trades_nats_v3
WHERE token_address = {token:String}
ORDER BY tx_timestamp DESC
LIMIT 100
FORMAT JSONCompact;

-- Expected: < 30ms
```

#### Pattern 3: 24h Volume
```sql
-- Option A: Query raw table (simple, works well)
SELECT
    sum(trade_total_usd) AS volume_24h,
    sumIf(trade_total_usd, trade_type = 'buy') AS buy_volume,
    sumIf(trade_total_usd, trade_type = 'sell') AS sell_volume,
    count() AS trade_count
FROM trades_nats_v3
WHERE token_address = {token:String}
  AND tx_timestamp >= now() - INTERVAL 24 HOUR;

-- Option B: Use materialized view (faster for hot tokens)
SELECT
    sum(volume_usd) AS volume_24h,
    sum(buy_volume_usd) AS buy_volume,
    sum(sell_volume_usd) AS sell_volume
FROM mv_volume_hourly
WHERE token_address = {token:String}
  AND hour >= now() - INTERVAL 24 HOUR;

-- Expected: < 100ms
```

#### Pattern 4: Top Holders
```sql
-- Uses mv_holder_balances for efficiency
SELECT
    wallet_address,
    sum(balance_change) AS balance
FROM mv_holder_balances
WHERE token_address = {token:String}
GROUP BY wallet_address
HAVING balance > 0
ORDER BY balance DESC
LIMIT 30
FORMAT JSONCompact;

-- Expected: < 150ms
```

#### Pattern 5: User Transaction History
```sql
SELECT
    tx_timestamp,
    token_address,
    trade_type,
    trade_total_usd,
    tx_signature
FROM trades_nats_v3
WHERE trader_wallet = {wallet:String}
ORDER BY tx_timestamp DESC
LIMIT 100
FORMAT JSONCompact;

-- Note: May need secondary index on trader_wallet for large datasets
```

### 5.2 Index Recommendations

```sql
-- Secondary index for wallet-based queries
ALTER TABLE trades_nats_v3
    ADD INDEX idx_trader_wallet (trader_wallet) TYPE bloom_filter GRANULARITY 4;

-- Secondary index for price range queries
ALTER TABLE trades_nats_v3
    ADD INDEX idx_price (price_usd) TYPE minmax GRANULARITY 4;

-- Skip index for text search (if needed)
ALTER TABLE tokens_nats_v2
    ADD INDEX idx_token_name (token_name) TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4;
```

### 5.3 Query Settings

```sql
-- Recommended session settings for API queries
SET max_threads = 4;                    -- Limit parallelism per query
SET max_memory_usage = 1000000000;      -- 1GB limit
SET max_execution_time = 30;            -- 30 second timeout
SET use_uncompressed_cache = 1;         -- Enable cache
SET load_balancing = 'nearest_hostname'; -- For cluster
```

---

## 6. Storage Management

### 6.1 Tiered Storage Configuration

```xml
<!-- storage.xml -->
<storage_configuration>
    <disks>
        <nvme>
            <path>/var/lib/clickhouse/</path>
        </nvme>
        <s3_cold>
            <type>s3</type>
            <endpoint>https://s3.us-east-1.amazonaws.com/click-clickhouse-cold/</endpoint>
            <access_key_id>...</access_key_id>
            <secret_access_key>...</secret_access_key>
        </s3_cold>
    </disks>
    <policies>
        <tiered>
            <volumes>
                <hot>
                    <disk>nvme</disk>
                </hot>
                <cold>
                    <disk>s3_cold</disk>
                </cold>
            </volumes>
            <move_factor>0.1</move_factor>
        </tiered>
    </policies>
</storage_configuration>
```

### 6.2 TTL Configuration

```sql
-- Hot tier: 30 days on NVMe, then move to S3
ALTER TABLE trades_nats_v3
    MODIFY TTL tx_timestamp + INTERVAL 30 DAY TO VOLUME 'cold';

-- Delete old data (if not needed)
ALTER TABLE token_metrics_nats_v2
    MODIFY TTL snapshot_timestamp + INTERVAL 7 DAY DELETE;
```

### 6.3 Compression Settings

```sql
-- Per-column compression for optimal ratio
ALTER TABLE trades_nats_v3
    MODIFY COLUMN token_address String CODEC(LZ4),
    MODIFY COLUMN trader_wallet String CODEC(LZ4),
    MODIFY COLUMN trade_total_usd Decimal64(6) CODEC(DoubleDelta, LZ4),
    MODIFY COLUMN tx_timestamp DateTime64(3) CODEC(DoubleDelta, LZ4);
```

**Compression Recommendations:**

| Data Type | Codec | Compression Ratio |
|-----------|-------|-------------------|
| Strings (addresses) | LZ4 | 3-5x |
| Timestamps | DoubleDelta + LZ4 | 10-20x |
| Decimals (prices) | DoubleDelta + LZ4 | 5-10x |
| Enums | LZ4 | 10x+ |
| Integers | Delta + LZ4 | 5-10x |

---

## 7. Cluster Configuration

### 7.1 Recommended Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLICKHOUSE CLUSTER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Shard 1                          Shard 2                       │
│  ┌─────────────────────┐         ┌─────────────────────┐       │
│  │  Replica 1 (Primary)│         │  Replica 1 (Primary)│       │
│  │  - NVMe storage     │         │  - NVMe storage     │       │
│  │  - Hot data         │         │  - Hot data         │       │
│  └─────────────────────┘         └─────────────────────┘       │
│           │                               │                     │
│           │ (Sync replication)            │                     │
│           ▼                               ▼                     │
│  ┌─────────────────────┐         ┌─────────────────────┐       │
│  │  Replica 2          │         │  Replica 2          │       │
│  │  - NVMe storage     │         │  - NVMe storage     │       │
│  │  - Hot data         │         │  - Hot data         │       │
│  └─────────────────────┘         └─────────────────────┘       │
│                                                                  │
│  Shared Cold Storage (S3)                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Historical data (> 30 days)                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Sharding Strategy

```xml
<!-- Shard by token_address for even distribution -->
<remote_servers>
    <click_cluster>
        <shard>
            <weight>1</weight>
            <internal_replication>true</internal_replication>
            <replica>
                <host>click-ch-1a</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>click-ch-1b</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <weight>1</weight>
            <internal_replication>true</internal_replication>
            <replica>
                <host>click-ch-2a</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>click-ch-2b</host>
                <port>9000</port>
            </replica>
        </shard>
    </click_cluster>
</remote_servers>
```

### 7.3 Distributed Tables

```sql
-- Create distributed table for queries
CREATE TABLE trades_distributed AS trades_nats_v3
ENGINE = Distributed(
    'click_cluster',
    'default',
    'trades_nats_v3',
    sipHash64(token_address)  -- Shard key
);
```

---

## 8. Monitoring & Observability

### 8.1 Key Metrics to Track

| Metric | Query | Alert Threshold |
|--------|-------|-----------------|
| **Ingestion Lag** | `max(now() - max(tx_timestamp))` | > 5 seconds |
| **Insert Rate** | `system.events` InsertedRows | < 50K/sec |
| **Query Latency** | `system.query_log` | p95 > 500ms |
| **Merge Lag** | `system.merges` | > 100 parts |
| **Disk Usage** | `system.disks` | > 80% |
| **Memory Usage** | `system.asynchronous_metrics` | > 80% |
| **Replication Lag** | `system.replicas` | > 10 seconds |

### 8.2 Prometheus Metrics

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'clickhouse'
    static_configs:
      - targets: ['click-ch-1a:9363', 'click-ch-1b:9363']
    metrics_path: '/metrics'
```

### 8.3 Grafana Dashboard Panels

**Essential Panels:**
1. Ingestion rate (rows/sec by topic)
2. Query latency distribution (p50, p95, p99)
3. Active queries count
4. Disk space by tier (hot vs cold)
5. Replication queue size
6. Parts count per table
7. Memory usage
8. Network throughput

### 8.4 Health Check Query

```sql
-- Run periodically to verify cluster health
SELECT
    database,
    table,
    count() AS partition_count,
    sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    max(modification_time) AS last_modified
FROM system.parts
WHERE active AND database = 'default'
GROUP BY database, table
ORDER BY total_rows DESC;
```

---

## 9. Backup & Recovery

### 9.1 Backup Strategy

```bash
# Full backup to S3 (weekly)
clickhouse-backup create --tables='default.*' weekly_$(date +%Y%m%d)
clickhouse-backup upload weekly_$(date +%Y%m%d)

# Incremental backup (daily)
clickhouse-backup create_remote --diff-from-remote=weekly_$(date -d 'last sunday' +%Y%m%d) daily_$(date +%Y%m%d)
```

### 9.2 Point-in-Time Recovery

```sql
-- ClickHouse supports ATTACH PARTITION for recovery
ALTER TABLE trades_nats_v3
    ATTACH PARTITION '20250115'
    FROM 'backup_trades_nats_v3';
```

---

## 10. Performance Benchmarks

### 10.1 Target Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Ingestion** | 100K+ events/sec | Sustained, all topics |
| **Ingestion Lag** | p95 < 1 second | From NATS publish to queryable |
| **OHLCV Query** | p95 < 50ms | 1000 candles |
| **Trade History** | p95 < 30ms | 100 rows |
| **Volume Aggregation** | p95 < 100ms | 24h window |
| **Holder Query** | p95 < 150ms | Top 30 |
| **User History** | p95 < 200ms | 100 rows |

### 10.2 Load Testing

```bash
# Use clickhouse-benchmark for query testing
clickhouse-benchmark \
    --host=click-ch-1a \
    --query="SELECT * FROM trades_nats_v3 WHERE token_address = 'ABC123' LIMIT 100" \
    --concurrency=50 \
    --iterations=1000

# Expected output:
# QPS: 500+
# p50: 20ms
# p95: 50ms
# p99: 100ms
```

---

## 11. Implementation Roadmap

### Phase 1: Core Tables (Week 1-2)
- [ ] Deploy ClickHouse cluster (2 shards, 2 replicas each)
- [ ] Create core table schemas
- [ ] Set up NATS consumers for existing topics
- [ ] Implement RowBinary encoding
- [ ] Configure tiered storage (NVMe + S3)
- [ ] Validate ingestion throughput (target: 100K/sec)

### Phase 2: Query Optimization (Week 3)
- [ ] Create materialized views for common aggregations
- [ ] Add secondary indexes
- [ ] Tune query settings
- [ ] Benchmark all query patterns
- [ ] Integrate with REST API

### Phase 3: Monitoring (Week 4)
- [ ] Set up Prometheus metrics export
- [ ] Create Grafana dashboards
- [ ] Configure alerts
- [ ] Document runbooks

### Phase 4: Hyperliquid Tables (Week 5-6)
- [ ] Create HL table schemas
- [ ] Set up Bincode → RowBinary converter
- [ ] Implement HL-specific ingestion
- [ ] Validate HL query patterns

### Phase 5: Production Hardening (Week 7-8)
- [ ] Implement backup automation
- [ ] Test failover scenarios
- [ ] Performance tuning
- [ ] Documentation

---

## 12. Troubleshooting Guide

### Common Issues

#### 1. High Ingestion Lag
```sql
-- Check part count (too many = not merging)
SELECT table, count() AS parts FROM system.parts WHERE active GROUP BY table;

-- Solution: Increase merge threads
ALTER TABLE trades_nats_v3 MODIFY SETTING max_parts_in_total = 100000;
```

#### 2. Slow Queries
```sql
-- Check query log for slow queries
SELECT
    query,
    query_duration_ms,
    read_rows,
    read_bytes
FROM system.query_log
WHERE query_duration_ms > 1000
ORDER BY query_start_time DESC
LIMIT 10;

-- Check if using wrong order/partition
EXPLAIN SELECT ... -- Look for "Parts: X" in output
```

#### 3. Out of Memory
```sql
-- Find memory-heavy queries
SELECT
    query,
    memory_usage,
    peak_memory_usage
FROM system.query_log
WHERE peak_memory_usage > 1000000000
ORDER BY event_time DESC;

-- Solution: Add LIMIT, reduce GROUP BY cardinality
```

#### 4. Replication Issues
```sql
-- Check replication queue
SELECT
    database,
    table,
    replica_name,
    queue_size,
    inserts_in_queue,
    merges_in_queue
FROM system.replicas
WHERE queue_size > 0;
```

---

## 13. Security Considerations

### 13.1 Access Control
```sql
-- Create read-only user for API
CREATE USER api_reader IDENTIFIED BY 'secure_password';
GRANT SELECT ON default.* TO api_reader;

-- Create writer user for ingestion
CREATE USER ingester IDENTIFIED BY 'secure_password';
GRANT INSERT ON default.* TO ingester;
```

### 13.2 Network Security
- ClickHouse ports (9000, 8123) should not be public
- Use VPN/private network for all connections
- Enable TLS for inter-node communication

### 13.3 Data Privacy
- No PII stored in ClickHouse (wallet addresses are public)
- User IDs are UUIDs (no correlation to identity)
- Access logs retained for audit

---

## 14. Cost Optimization

### 14.1 Storage Cost Breakdown

| Tier | Storage | Cost | Data Age |
|------|---------|------|----------|
| Hot (NVMe) | 2TB | ~$400/month | 0-30 days |
| Cold (S3) | 20TB | ~$400/month | 30+ days |
| **Total** | 22TB | ~$800/month | - |

### 14.2 Cost Reduction Strategies

1. **Aggressive TTL**: Delete metrics older than 7 days
2. **Compression**: Use optimal codecs (50% size reduction)
3. **Tiered Storage**: Move data to S3 after 30 days
4. **Query Caching**: Reduce compute for repeated queries
5. **Materialized Views**: Pre-compute expensive aggregations

---

## Quick Reference Card

### Connection
```
Host: click-ch-1a (or load balancer)
Port: 9000 (native), 8123 (HTTP)
Database: default
```

### Common Queries
```sql
-- Latest price
SELECT price_usd FROM trades_nats_v3
WHERE token_address = ? ORDER BY tx_timestamp DESC LIMIT 1;

-- 24h volume
SELECT sum(trade_total_usd) FROM trades_nats_v3
WHERE token_address = ? AND tx_timestamp > now() - INTERVAL 24 HOUR;

-- Top holders
SELECT wallet_address, sum(balance_change) AS bal FROM mv_holder_balances
WHERE token_address = ? GROUP BY wallet_address HAVING bal > 0 ORDER BY bal DESC LIMIT 30;
```

### Useful System Queries
```sql
-- Table sizes
SELECT table, formatReadableSize(sum(bytes)) FROM system.parts GROUP BY table;

-- Active queries
SELECT query_id, query, elapsed FROM system.processes;

-- Kill query
KILL QUERY WHERE query_id = '...';
```

---

## 15. Advanced Optimization Guide (From ClickHouse Blogs)

This section compiles all optimization recommendations from official ClickHouse blogs and community best practices.

### 15.1 The "13 Deadly Sins" - Common Mistakes to Avoid

Based on [ClickHouse's official blog on common issues](https://clickhouse.com/blog/common-getting-started-issues-with-clickhouse):

#### Sin #1: Too Many Parts
**Problem**: As the number of parts increases, queries slow down due to evaluating more indices and reading more files.

**Causes**:
- Using partition key with excessive cardinality
- Small batch inserts creating many parts
- Insufficient merge thread capacity

**Solution**:
```sql
-- Check part count per table
SELECT database, table, count() AS parts
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY parts DESC;

-- Healthy tables should have < 100 active parts
-- If > 300 parts per partition, inserts will be rejected
```

#### Sin #2: Excessive Mutations
**Problem**: ClickHouse performs best on immutable data. Mutations rewrite entire data parts.

**Solution**: Design for append-only patterns. Use ReplacingMergeTree for updates instead of UPDATE statements.

#### Sin #3: Wrong Primary Key Cardinality Order
**Problem**: Sorting by high-cardinality column first (like timestamp) leads to near-complete table scans.

**Why**:
- First column uses binary search O(log₂ n) - efficient regardless of cardinality
- Secondary columns use generic exclusion search - depends on predecessor's cardinality
- Low predecessor cardinality = effective filtering (same value spans multiple granules)
- High predecessor cardinality = ineffective filtering

**Rule**: Put LOW cardinality columns FIRST in ORDER BY.

```sql
-- BAD: High cardinality first
ORDER BY (tx_timestamp, token_address)

-- GOOD: Low cardinality first
ORDER BY (token_address, tx_timestamp)
```

#### Sin #4: Not Including WHERE Columns in Primary Key
**Problem**: If query filters on column not in primary key, ClickHouse must scan every granule.

```sql
-- If table has ORDER BY (postcode, address) but query filters:
WHERE town = 'LONDON'  -- Full table scan!

-- Solution: Include frequently filtered columns in ORDER BY
ORDER BY (town, postcode, address)
```

#### Sin #5: Modeling Joins Like PostgreSQL
**Problem**: ClickHouse JOINs can be expensive. Pre-aggregated/denormalized data performs best.

**Solution**: Use materialized views to denormalize at insert time.

#### Sin #6: Using SELECT *
**Problem**: ClickHouse is columnar - fetching unnecessary columns wastes I/O.

**Solution**: Always specify only needed columns.

#### Sin #7: Small Batch Inserts
**Problem**: Each insert creates a new part. Too many small parts = merge pressure.

**Solution**: Batch 10K-100K rows per insert. Never insert row-by-row.

#### Sin #8: Nullable Columns Overuse
**Problem**: Nullable columns add an extra bitmask, increasing processing overhead.

**Solution**: Use empty strings or 0 for missing values when possible.

---

### 15.2 Primary Key Optimization

Based on [ClickHouse's definitive query optimization guide](https://clickhouse.com/resources/engineering/clickhouse-query-optimisation-definitive-guide):

#### The Granule Ratio Metric
**Key insight**: The granule ratio tells you how well your primary key is working.

```sql
-- Check granule efficiency with EXPLAIN
EXPLAIN indexes = 1
SELECT count() FROM trades_nats_v3 WHERE token_address = 'ABC123';

-- Look for: "Selected X/Y granules"
-- If X ≈ Y (all granules selected) = full table scan = bad primary key
```

#### Primary Key Design Rules

| Rule | Description |
|------|-------------|
| Include WHERE columns | Columns used in filters should be in ORDER BY |
| Low cardinality first | Put columns with fewer unique values first |
| Time component | Include time for time-series data |
| Can't change later | No ALTER TABLE for sort keys - must rebuild |

#### Optimal ORDER BY for Click Tables

```sql
-- trades_nats_v3: Most queries filter by token, then time
ORDER BY (token_address, tx_timestamp, tx_signature)
-- token_address: ~100K unique (lower cardinality)
-- tx_timestamp: millions (higher cardinality)

-- For user history queries, consider a projection:
ALTER TABLE trades_nats_v3 ADD PROJECTION user_trades (
    SELECT * ORDER BY (trader_wallet, tx_timestamp)
);
```

---

### 15.3 PREWHERE Optimization

Based on [ClickHouse PREWHERE documentation](https://clickhouse.com/docs/optimize/prewhere):

#### How PREWHERE Works
PREWHERE filters data BEFORE reading non-filter columns, reducing I/O dramatically.

**Example**: Query with 3 columns where filter matches 10% of rows:
- Without PREWHERE: Read all 3 columns for all rows
- With PREWHERE: Read filter column, then read other 2 columns for 10% of rows only

#### Automatic Optimization (Enabled by Default)
```sql
-- ClickHouse automatically moves WHERE to PREWHERE when beneficial
-- Controlled by: optimize_move_to_prewhere = 1 (default)

-- Verify it's being applied:
EXPLAIN SYNTAX SELECT * FROM trades WHERE token_address = 'ABC';
-- Output will show PREWHERE if optimized
```

#### Multi-Step PREWHERE (v23.2+)
```sql
-- Enable for better performance with multiple filter columns
SET enable_multiple_prewhere_read_steps = 1;
SET move_all_conditions_to_prewhere = 1;

-- ClickHouse will process filters in order of column size (smallest first)
```

#### Performance Impact
- Typical: 3x less data read (e.g., 6.74 MB instead of 23.36 MB)
- Query time reduction: 30-40% on filter-heavy queries

---

### 15.4 Skip Indexes (Data Skipping Indexes)

Based on [ClickHouse skip index documentation](https://clickhouse.com/docs/optimize/skipping-indexes/examples) and [Altinity's guide](https://altinity.com/blog/clickhouse-black-magic-skipping-indices):

#### Index Types

| Type | Best For | How It Works |
|------|----------|--------------|
| **minmax** | Range queries, sorted data | Stores min/max per block, skips if range doesn't match |
| **bloom_filter** | Equality checks | Probabilistic - 100% accurate for negatives |
| **set(N)** | Low cardinality columns | Stores unique values per block |
| **ngrambf_v1** | LIKE '%pattern%' searches | Splits strings into n-grams |
| **tokenbf_v1** | Word-based text search | Tokenizes strings |

#### MinMax Index Example
```sql
-- Great for sorted/semi-sorted columns
ALTER TABLE trades_nats_v3
    ADD INDEX idx_timestamp_minmax (tx_timestamp)
    TYPE minmax
    GRANULARITY 1;

-- Must materialize for existing data
ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_timestamp_minmax;
```

#### Bloom Filter for Wallet Lookups
```sql
-- Bloom filters excel at equality checks on high-cardinality columns
ALTER TABLE trades_nats_v3
    ADD INDEX idx_wallet_bloom (trader_wallet)
    TYPE bloom_filter(0.01)  -- 1% false positive rate
    GRANULARITY 4;

ALTER TABLE trades_nats_v3 MATERIALIZE INDEX idx_wallet_bloom;
```

#### Key Rules for Skip Indexes

1. **Don't over-index**: Every index adds storage and slows inserts
2. **Test granularity**: GRANULARITY 1 vs 4 vs 8 - test on your data
3. **Skip indexes should skip**: If every granule is positive, it's wasting CPU
4. **Don't index primary key columns**: Primary key already provides filtering
5. **Bloom filters don't work with NOT**: Can't use for `!=` or `NOT LIKE`

#### Verify Index Usage
```sql
SET send_logs_level = 'trace';
SELECT count() FROM trades WHERE trader_wallet = 'ABC123';
-- Check logs for "Index ... dropped X/Y granules"
```

---

### 15.5 Compression Codec Optimization

Based on [ClickHouse compression blog](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema):

#### Codec Selection Guide

| Data Type | Recommended Codec | Why |
|-----------|-------------------|-----|
| **DateTime/Timestamps** | `DoubleDelta, ZSTD(1)` | Timestamps are sequential |
| **Monotonic integers** | `Delta, LZ4` | IDs, slots, sequence numbers |
| **Prices (Decimal)** | `Gorilla, ZSTD(1)` or `DoubleDelta, LZ4` | Financial data patterns |
| **Strings (addresses)** | `LZ4` | Fast compression, decent ratio |
| **Low cardinality strings** | `LowCardinality(String)` | Better than CODEC |
| **Boolean/flags** | `ZSTD(1)` | Small data, high compression |

#### Codec Chaining
```sql
-- Codecs are applied in order: Delta -> then ZSTD
CREATE TABLE example (
    ts DateTime64(3) CODEC(DoubleDelta, ZSTD(1)),
    price Decimal64(8) CODEC(Gorilla, ZSTD(1)),
    slot UInt64 CODEC(Delta, LZ4),
    token_address String CODEC(LZ4),
    dex_source LowCardinality(String)  -- No CODEC needed
);
```

#### Compression Level Guidance
- **ZSTD(1)**: Default, good balance
- **ZSTD(3)**: Slightly better compression, rarely worth CPU cost
- **ZSTD levels > 3**: Rarely significant gains
- **LZ4**: Faster decompression, use for hot data

#### LowCardinality vs CODEC
```sql
-- For columns with < 10K unique values, use LowCardinality
dex_source LowCardinality(String)    -- GOOD: ~20 unique DEXes
token_address String CODEC(LZ4)       -- GOOD: 100K+ unique tokens

-- LowCardinality is NOT a codec - it's a data type wrapper
```

---

### 15.6 Insert Optimization

Based on [ClickHouse supercharge data loads blog](https://clickhouse.com/blog/supercharge-your-clickhouse-data-loads-part1) and [async inserts guide](https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse):

#### Batch Size Recommendations

| Scenario | Batch Size | Flush Interval |
|----------|------------|----------------|
| High throughput | 50K-100K rows | 5-10 seconds |
| Low latency | 10K-50K rows | 1-5 seconds |
| Real-time | 1K-10K rows | 0.5-2 seconds |

**Key insight**: Batching is the single most important optimization. Mechanics show constant overhead per insert regardless of size.

#### Async Inserts Configuration
```sql
-- Enable async inserts for high-concurrency scenarios
SET async_insert = 1;
SET wait_for_async_insert = 1;  -- CRITICAL: Always 1 in production!
SET async_insert_max_data_size = 10000000;  -- 10MB buffer
SET async_insert_busy_timeout_ms = 2000;    -- 2 second timeout
SET async_insert_max_query_number = 450;    -- Max queries in buffer
```

**CRITICAL WARNING**: Never use `wait_for_async_insert = 0` in production. Teams have lost millions of records from this mistake.

#### When to Use Async Inserts
- Many independent small inserts (IoT, logging)
- Can't batch on client side
- NOT when you need immediate read-after-write

#### Format Performance
```
Native:       ~200K rows/sec (best throughput)
RowBinary:    ~150K rows/sec (recommended)
JSONEachRow:  ~50K rows/sec  (simple pipelines)
JSON:         ~30K rows/sec  (development only)
```

---

### 15.7 Partitioning Best Practices

Based on [ClickHouse partitioning guide](https://clickhouse.com/docs/best-practices/choosing-a-partitioning-key):

#### Golden Rule
**Partitioning is for data management, not query optimization.**

#### Partition Count Guidelines
- Target: Dozens to hundreds of partitions
- Never: Thousands of partitions
- Max per insert: 100 partitions (configurable)
- Max partition size: ~300 GB

#### Choosing Partition Key

| Data Volume | Recommended Strategy |
|-------------|---------------------|
| < 10 GB | No partitioning needed |
| 10-100 GB | Monthly: `toYYYYMM(timestamp)` |
| 100 GB - 1 TB | Weekly: `toMonday(timestamp)` |
| > 1 TB | Daily: `toYYYYMMDD(timestamp)` |

#### Avoid These Mistakes
```sql
-- BAD: High cardinality partition key
PARTITION BY token_address  -- Could be 100K+ partitions!

-- BAD: Too granular for data volume
PARTITION BY toYYYYMMDD(timestamp)  -- If only 1GB/month

-- GOOD: Match partition size to data volume
PARTITION BY toYYYYMM(timestamp)  -- Monthly for moderate volume
```

#### Click-Specific Recommendations
```sql
-- trades_nats_v3: High volume, daily partitions
PARTITION BY toYYYYMMDD(tx_timestamp)

-- tokens_nats_v2: Low volume, monthly partitions
PARTITION BY toYYYYMM(deployment_timestamp)

-- ohlcv_candles: Composite partition for query patterns
PARTITION BY (timeframe, toYYYYMM(candle_timestamp))
```

---

### 15.8 Materialized Views vs Projections

Based on [ClickHouse MV vs Projections guide](https://clickhouse.com/docs/managing-data/materialized-views-versus-projections):

#### When to Use Materialized Views
- Multi-stage ETL pipelines
- Complex denormalization with JOINs
- Different retention policies
- Aggregations across tables

```sql
-- Materialized View: Creates separate table
CREATE MATERIALIZED VIEW mv_volume_hourly
ENGINE = SummingMergeTree()
ORDER BY (token_address, hour)
AS SELECT
    token_address,
    toStartOfHour(tx_timestamp) AS hour,
    sum(trade_total_usd) AS volume_usd
FROM trades_nats_v3
GROUP BY token_address, hour;
```

#### When to Use Projections
- Alternative sort orders for single table
- Query transparency (auto-selected)
- No JOINs or complex transforms

```sql
-- Projection: Stored within same table
ALTER TABLE trades_nats_v3 ADD PROJECTION proj_by_wallet (
    SELECT * ORDER BY (trader_wallet, tx_timestamp)
);
ALTER TABLE trades_nats_v3 MATERIALIZE PROJECTION proj_by_wallet;

-- Queries automatically use projection when beneficial
SELECT * FROM trades_nats_v3
WHERE trader_wallet = 'ABC'
ORDER BY tx_timestamp DESC;
```

#### Limitations of Projections
- Can't use JOINs in definition
- Can't filter data (no WHERE in definition)
- Don't work with FINAL queries
- Don't work with parallel replicas

#### Best Practice
Use BOTH complementarily:
- Projections for alternative sort orders
- Materialized views for aggregations and denormalization

---

### 15.9 Memory Management

Based on [ClickHouse memory configuration guide](https://kb.altinity.com/altinity-kb-setup-and-maintenance/altinity-kb-memory-configuration-settings/):

#### Key Memory Settings

| Setting | Default | Recommendation |
|---------|---------|----------------|
| `max_memory_usage` | 10 GB | 70-80% RAM / concurrent queries |
| `max_server_memory_usage` | 90% RAM | Leave 10% for OS |
| `max_bytes_before_external_sort` | 0 (disabled) | Set to 50% of max_memory_usage |
| `max_bytes_before_external_group_by` | 0 (disabled) | Set to 50% of max_memory_usage |

#### Enable Disk Spilling
```sql
-- Allow large queries to spill to disk instead of OOM
SET max_bytes_before_external_group_by = 5000000000;  -- 5GB
SET max_bytes_before_external_sort = 5000000000;      -- 5GB
```

#### RAM Guidelines by Data Volume
- < 200 GB data: RAM = data volume (fit in memory)
- > 200 GB data: 128 GB+ RAM for hot data in page cache
- 50 TB data: 128 GB RAM significantly better than 64 GB

#### Low Memory Environments (4-8 GB)
```sql
-- Conservative settings for small instances
SET max_memory_usage = 2000000000;           -- 2GB per query
SET max_concurrent_queries = 8;
SET max_threads = 2;
SET mark_cache_size = 268435456;             -- 256MB
SET uncompressed_cache_size = 16777216;      -- 16MB
```

---

### 15.10 Approximate Functions for Speed

Based on [ClickHouse query optimization guide](https://clickhouse.com/blog/a-simple-guide-to-clickhouse-query-optimization-part-1):

#### When to Use Approximate Functions

| Exact Function | Approximate Function | Speed Improvement |
|----------------|---------------------|-------------------|
| `COUNT(DISTINCT x)` | `uniq(x)` | 10-100x faster |
| `COUNT(DISTINCT x)` | `uniqCombined(x)` | Even faster, less accurate |
| Percentile | `quantile(x)` | Much faster |
| Median | `median(x)` | Uses approximate quantile |

```sql
-- Exact (slow for large datasets)
SELECT COUNT(DISTINCT trader_wallet) FROM trades_nats_v3;

-- Approximate (10-100x faster, ~2% error)
SELECT uniq(trader_wallet) FROM trades_nats_v3;

-- For dashboards where exact count isn't critical
SELECT uniqCombined(trader_wallet) FROM trades_nats_v3;
```

#### Click Use Cases for Approximation
- Discovery page: Unique trader counts (use `uniq`)
- Token metrics: Holder counts (use `uniq`)
- Volume charts: Trade counts (exact is fine, fast anyway)

---

### 15.11 Query Analysis & Debugging

#### Using EXPLAIN
```sql
-- See query plan
EXPLAIN SELECT * FROM trades WHERE token_address = 'ABC';

-- See with index info
EXPLAIN indexes = 1 SELECT * FROM trades WHERE token_address = 'ABC';

-- See actual syntax after optimization
EXPLAIN SYNTAX SELECT * FROM trades WHERE token_address = 'ABC';

-- Full analysis
EXPLAIN AST SELECT * FROM trades WHERE token_address = 'ABC';
```

#### Query Log Analysis
```sql
-- Find slow queries
SELECT
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage,
    ProfileEvents['SelectedMarks'] AS marks_selected,
    ProfileEvents['SelectedRows'] AS rows_selected
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000
ORDER BY query_start_time DESC
LIMIT 20;
```

#### Identify Full Table Scans
```sql
-- Queries reading too much data
SELECT
    query,
    read_rows,
    read_bytes,
    result_rows,
    read_rows / result_rows AS amplification_factor
FROM system.query_log
WHERE type = 'QueryFinish'
  AND read_rows > 1000000
  AND result_rows < 1000
ORDER BY amplification_factor DESC
LIMIT 20;
```

---

### 15.12 Optimization Checklist

Use this checklist before deploying new tables:

#### Schema Design
- [ ] ORDER BY includes frequently filtered columns
- [ ] Low cardinality columns come first in ORDER BY
- [ ] Using LowCardinality for columns with < 10K unique values
- [ ] Avoided Nullable where possible
- [ ] Applied appropriate compression codecs

#### Partitioning
- [ ] Partition key has low cardinality (dozens/hundreds, not thousands)
- [ ] Partition size is reasonable (not too small, not > 300GB)
- [ ] Time-based partitioning matches data volume

#### Ingestion
- [ ] Batch size is 10K-100K rows
- [ ] Using RowBinary or Native format
- [ ] Async inserts configured with `wait_for_async_insert = 1`

#### Query Optimization
- [ ] Skip indexes added for non-primary-key filters
- [ ] Materialized views for common aggregations
- [ ] Projections for alternative access patterns
- [ ] Using approximate functions where appropriate

#### Monitoring
- [ ] Part count alerts configured (< 100 per table)
- [ ] Query latency tracking enabled
- [ ] Memory usage alerts set

---

## Sources

This optimization guide compiled from:

- [ClickHouse: The definitive guide to query optimization (2026)](https://clickhouse.com/resources/engineering/clickhouse-query-optimisation-definitive-guide)
- [ClickHouse: A simple guide to query optimization - Part 1](https://clickhouse.com/blog/a-simple-guide-to-clickhouse-query-optimization-part-1)
- [ClickHouse: Common getting started issues ("13 Deadly Sins")](https://clickhouse.com/blog/common-getting-started-issues-with-clickhouse)
- [ClickHouse: Supercharging large data loads - Part 1](https://clickhouse.com/blog/supercharge-your-clickhouse-data-loads-part1)
- [ClickHouse: Asynchronous data inserts](https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse)
- [ClickHouse: Optimize with codecs and compression](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema)
- [ClickHouse: Super charging queries with projections](https://clickhouse.com/blog/clickhouse-faster-queries-with-projections-and-primary-indexes)
- [ClickHouse: PREWHERE optimization docs](https://clickhouse.com/docs/optimize/prewhere)
- [ClickHouse: Skip indexes documentation](https://clickhouse.com/docs/optimize/skipping-indexes/examples)
- [ClickHouse: Choosing a partitioning key](https://clickhouse.com/docs/best-practices/choosing-a-partitioning-key)
- [ClickHouse: Materialized views vs projections](https://clickhouse.com/docs/managing-data/materialized-views-versus-projections)
- [Altinity: Skip indexes black magic](https://altinity.com/blog/clickhouse-black-magic-skipping-indices)
- [Altinity: Memory configuration settings](https://kb.altinity.com/altinity-kb-setup-and-maintenance/altinity-kb-memory-configuration-settings/)
- [Highlight.io: Optimizing ClickHouse - tactics that worked](https://www.highlight.io/blog/lw5-clickhouse-performance-optimization)
- [e6data: ClickHouse query performance optimization 2025](https://www.e6data.com/query-and-cost-optimization-hub/how-to-optimize-clickhouse-query-performance)
