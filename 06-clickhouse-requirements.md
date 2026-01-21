# ClickHouse Requirements for Click

## Overview

ClickHouse is Click's analytical database for storing and querying historical market data. It is the **permanent storage layer** that receives pre-computed data from DataStreams via NATS.

**Key Point**: ClickHouse does NOT compute metrics. DataStreams computes everything, ClickHouse stores and serves.

---

## Role in Architecture

```
Solana Geyser → Solana Engine → NATS → DataStreams → NATS → ClickHouse
                                                          ↓
                                                    REST API (queries)
```

### What ClickHouse Does
- Ingests all events from NATS
- Stores permanently (hot + cold tiers)
- Serves analytical queries

### What ClickHouse Does NOT Do
- Real-time metric computation (DataStreams)
- User data storage (Postgres)
- Message routing (NATS)
- Publishing data (DataStreams)

---

## NATS Topics to Ingest

### From Solana Engine

| Topic | Event Type | Format |
|-------|------------|--------|
| `solana.trades.v3` | Swap | RowBinary |
| `solana.transfers.v2` | Transfer | RowBinary |
| `solana.tokens.v2` | Token | RowBinary |
| `solana.pools.v1_0_0` | Pool | RowBinary |
| `solana.liquidity_events.v1` | LiquidityEvent | RowBinary |
| `solana.token_events.v1` | TokenEvent | RowBinary |
| `solana.graduation_events.v1` | GraduationEvent | RowBinary |
| `solana.failed_transactions.v1` | FailedTransaction | RowBinary |

### From Hyperliquid Indexer

| Topic | Event Type | Format |
|-------|------------|--------|
| `hyperliquid.fills` | HyperliquidFill | Bincode |
| `hyperliquid.deposits` | HyperliquidDeposit | Bincode |
| `hyperliquid.withdrawals` | HyperliquidWithdrawal | Bincode |
| `hyperliquid.funding` | HyperliquidFunding | Bincode |
| `hyperliquid.liquidations` | HyperliquidLiquidation | Bincode |

### From DataStreams (Derived Data)

| Topic | Event Type | Format |
|-------|------------|--------|
| `solana.enriched_trades` | Trade | RowBinary |
| `solana.market_ohlcv.*` | Ohlcv | RowBinary |
| `solana.token_ohlcv.*` | Ohlcv | RowBinary |
| `solana.token_metrics.v2` | TokenMetrics | RowBinary |
| `solana.discovery.*` | Discovery data | RowBinary |
| `click.order_overview` | OrderOverview | RowBinary |
| `solana.position_overview` | PositionOverview | RowBinary |

---

## Current Tables (Existing)

| Table | NATS Subject | What It Captures |
|-------|--------------|------------------|
| `trades_nats_v3` | `solana.trades.v3` | Raw swap/trade events |
| `tokens_nats_v2` | `solana.tokens.v2` | Token metadata |
| `pools_nats_v1_0_0` | `solana.pools.v1_0_0` | Liquidity pool data |
| `transfers_nats_v2` | `solana.transfers.v2` | Token transfers |
| `token_metrics_nats_v2` | `solana.token_metrics.v2` | Real-time token metrics |

---

## Missing Tables (To Add)

### Priority 0 (Critical for Core Features)

| Table | NATS Subject | Purpose | Used By |
|-------|--------------|---------|---------|
| **OHLCV Candles** | `solana.token_ohlcv.*` | Historical charts (14 TF) | Token Trading Page |
| **Enriched Trades** | `solana.enriched_trades` | Trades with wallet badges | Trades Tab |
| **Discovery Data** | `solana.discovery.*` | Pre-computed discovery | Discovery Page |
| **Position Overview** | `solana.position_overview` | User position snapshots | Portfolio Page |
| **Order Overview** | `click.order_overview` | Order summaries | Orders Tab |

### Priority 1 (Required for Perps)

| Table | NATS Subject | Purpose | Used By |
|-------|--------------|---------|---------|
| **HL Fills** | `hyperliquid.fills` | Perps trade history | Perps Portfolio |
| **HL Funding** | `hyperliquid.funding` | Funding payments | Perps Positions |
| **HL Liquidations** | `hyperliquid.liquidations` | Liquidation events | Perps Risk |
| **HL Deposits** | `hyperliquid.deposits` | Deposit tracking | Perps Wallet |
| **HL Withdrawals** | `hyperliquid.withdrawals` | Withdrawal tracking | Perps Wallet |

### Priority 2 (Lifecycle Events)

| Table | NATS Subject | Purpose | Used By |
|-------|--------------|---------|---------|
| **Graduation Events** | `solana.graduation_events.v1` | Bonding curve graduation | Discovery |
| **Liquidity Events** | `solana.liquidity_events.v1` | LP create/burn | Chart markers |
| **Token Events** | `solana.token_events.v1` | Token lifecycle | Discovery |
| **Failed Transactions** | `solana.failed_transactions.v1` | Failed tx tracking | History Tab |

---

## Table Schemas

### OHLCV Candles

```sql
CREATE TABLE ohlcv_candles (
    token_address String,
    timeframe Enum8('1s'=1, '5s'=2, '15s'=3, '30s'=4,
                    '1m'=5, '5m'=6, '15m'=7, '30m'=8,
                    '1h'=9, '4h'=10, '6h'=11, '12h'=12,
                    '1d'=13, '1w'=14),
    candle_timestamp DateTime64(3),
    open Decimal64(18),
    high Decimal64(18),
    low Decimal64(18),
    close Decimal64(18),
    volume Decimal64(18)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(candle_timestamp)
ORDER BY (token_address, timeframe, candle_timestamp);
```

### Enriched Trades

```sql
CREATE TABLE enriched_trades (
    tx_signature String,
    token_address String,
    trade_type Enum8('buy'=1, 'sell'=2),
    trade_amount Decimal64(18),
    trade_total_usd Decimal64(18),
    trade_mcap_usd Decimal64(18),
    trader_wallet String,
    is_sniper UInt8,
    is_insider UInt8,
    is_bundler UInt8,
    is_dev UInt8,
    is_whale UInt8,
    is_kol UInt8,
    tx_timestamp DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(tx_timestamp)
ORDER BY (token_address, tx_timestamp, tx_signature);
```

### Hyperliquid Fills

```sql
CREATE TABLE hl_fills (
    fill_id String,
    user_address String,
    coin String,
    direction Enum8('long'=1, 'short'=2),
    size Decimal64(18),
    price Decimal64(18),
    fee Decimal64(18),
    realized_pnl Decimal64(18),
    timestamp DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (user_address, timestamp, fill_id);
```

---

## Performance Requirements

### Ingestion

| Metric | Target |
|--------|--------|
| Throughput | 100,000+ events/second |
| Ingestion lag | p95 < 1 second |
| Data loss | Zero (at-least-once with deduplication) |

### Query Performance

| Query Type | Latency Target |
|------------|----------------|
| OHLCV lookups | p95 < 400ms |
| Trade history | p95 < 400ms |
| Analytics aggregations | p95 < 400ms |
| Token metadata | p95 < 200ms |
| General analytical | p95 < 250ms |

### Availability

| Metric | Target |
|--------|--------|
| Uptime | 99.9% |
| Single-node failure | No data loss, no query interruption |
| Rolling upgrades | Zero downtime |

---

## Storage Architecture

### Tiered Storage

| Tier | Storage | Purpose | Data Age |
|------|---------|---------|----------|
| Hot | NVMe (local) | Low-latency queries | Recent (configurable) |
| Cold | S3 | Cost-efficient retention | Historical |

### Requirements
- Automatic migration based on age
- Transparent querying across tiers
- >50% cost reduction vs all-hot

### Retention
- Hot tier: Configurable window (e.g., 7-30 days)
- Cold tier: Long-term retention
- TTL policies at table level

---

## Query Patterns to Optimize

### 1. OHLCV Queries
```sql
-- Historical chart data
SELECT * FROM ohlcv_candles
WHERE token_address = ?
  AND timeframe = ?
  AND candle_timestamp BETWEEN ? AND ?
ORDER BY candle_timestamp;
```

### 2. Trade History by Token
```sql
-- Trades tab
SELECT * FROM trades_nats_v3
WHERE token_address = ?
ORDER BY tx_timestamp DESC
LIMIT 100;
```

### 3. Trade History by Wallet
```sql
-- User transaction history
SELECT * FROM trades_nats_v3
WHERE trader_wallet = ?
ORDER BY tx_timestamp DESC;
```

### 4. Volume Aggregations
```sql
-- 24h volume
SELECT SUM(trade_total_usd) as volume_24h
FROM trades_nats_v3
WHERE token_address = ?
  AND tx_timestamp > now() - INTERVAL 24 HOUR;
```

### 5. Holder Analytics
```sql
-- Top holders
SELECT
    wallet,
    SUM(CASE WHEN type = 'receive' THEN amount ELSE -amount END) as balance
FROM transfers_nats_v2
WHERE token_address = ?
GROUP BY wallet
HAVING balance > 0
ORDER BY balance DESC
LIMIT 30;
```

---

## Indexing Strategy

### Primary Keys
- `token_address` - Most queries filter by token
- `tx_timestamp` - Time-series queries
- Composite: `(token_address, timestamp)`

### Partitioning
- By date: `toYYYYMMDD(timestamp)` or `toYYYYMM(timestamp)`
- Enables efficient time-range queries
- Easy data lifecycle management

### Materialized Views
Consider for:
- Pre-aggregated volume metrics
- Holder balance rollups
- OHLCV if not from DataStreams

---

## Scaling Strategy

### Horizontal Scaling
- Sharding by `token_address`
- Distributes hot tokens across nodes

### Columnar Optimization
- Only read columns needed
- Compression per column type

### Connection Pooling
- Handle REST API query load
- Prevent connection exhaustion

---

## Monitoring & Observability

### Metrics to Track

| Metric | Purpose |
|--------|---------|
| Ingestion lag (by topic) | Data freshness |
| Events ingested/second (by source) | Throughput |
| Query latency (by query type) | Performance |
| Active connections | Load |
| Concurrent queries | Capacity |
| Storage utilization (hot vs cold) | Capacity planning |
| Replication lag | Cluster health |

### Dashboards
- Ingestion health (lag, throughput, errors)
- Query performance (latency distribution)
- Storage (utilization, tier breakdown)
- Cluster health (replication, node status)

---

## Metrics Derivable from Current Tables

### From `trades_nats_v3`
- Current price (latest trade)
- Price changes (5m, 1h, 6h, 24h)
- Volume (24h, 1h, by type)
- Buy/sell volume split
- Transaction counts
- Unique trader count
- OHLCV candles (via aggregation)
- Trade history

### From `tokens_nats_v2`
- Token metadata (name, symbol, image)
- Social links
- Deployment info
- Creator/dev wallet
- Supply

### From `pools_nats_v1_0_0`
- Liquidity
- Burned LP %
- Pool count
- Graduation status

### From `transfers_nats_v2`
- Holder count
- Top holder concentration
- Insider detection
- Dev holdings
- Wallet balances

### From `token_metrics_nats_v2`
- Pre-computed real-time metrics
- (Schema-dependent)

---

## Gap Analysis Summary

| Feature Area | Current Coverage | Gap |
|--------------|------------------|-----|
| Token Metadata | ✅ 100% | None |
| Price & Market Cap | ✅ 100% | None |
| Volume Metrics | ✅ 100% | None |
| Liquidity | ✅ 100% | None |
| Trade History | ✅ 100% | None |
| Holder Count | ✅ 100% | None |
| Basic Holder Analytics | ✅ ~80% | Bundler needs DataStreams |
| **OHLCV Charts** | ⚠️ Derivable | Add dedicated table for perf |
| **Wallet Classifications** | ⚠️ Partial | Need enriched_trades table |
| **Perps Data** | ❌ 0% | Need Hyperliquid tables |
| **Lifecycle Events** | ❌ 0% | Need graduation/liquidity tables |

---

## Implementation Priority

### Phase 1: Core Features
1. OHLCV Candles table (or optimize aggregation)
2. Enriched Trades table
3. Discovery data table
4. Position Overview table
5. Order Overview table

### Phase 2: Perps
1. Hyperliquid Fills
2. Hyperliquid Funding
3. Hyperliquid Liquidations

### Phase 3: Events & Lifecycle
1. Graduation Events
2. Liquidity Events
3. Failed Transactions
