# Crypto Data Engineering: Concepts & Terminology

## Overview

This document covers theoretical concepts and terminology for building crypto data infrastructure:

**Part 1: Crypto Market Fundamentals (Sections 1-4)**
1. Market structure & instrument types
2. Derivatives mechanics
3. Key metrics & indicators
4. Exchange ecosystem

**Part 2: Data Engineering Patterns (Sections 5-9)**
5. Data sourcing & aggregation
6. Pipeline architecture
7. Storage patterns
8. Data quality & anomaly handling
9. Reference data management

**Part 3: Glossary (Section 10)**
10. Complete terminology reference

---

## 1. Market Structure & Instrument Types

### 1.1 Spot Markets

**Definition**: Direct exchange of one asset for another at current market price.

```
Spot Trade: Buy 1 BTC for 45,000 USDT
- Immediate settlement
- You own the underlying asset
- No expiry, no leverage (unless margin)
```

**Key Characteristics**:
- Immediate ownership transfer
- Price = current market value
- No expiration
- Settlement: T+0 (instant in crypto)

**Data Points to Capture**:
| Field | Description | Example |
|-------|-------------|---------|
| `price` | Trade price | 45123.45 |
| `quantity` | Amount traded | 0.5 BTC |
| `side` | Buy or Sell | buy |
| `timestamp` | Trade time (ms) | 1705766400000 |
| `trade_id` | Exchange trade ID | 123456789 |

### 1.2 Perpetual Swaps (Perps)

**Definition**: Derivative contracts that track an underlying asset's price with no expiration date.

```
Perp Trade: Long 10 BTC-PERP at 45,000
- No expiry (unlike futures)
- Leverage: 1x to 100x+
- Funding rate mechanism keeps price anchored to spot
```

**How Perps Work**:
```
┌─────────────────────────────────────────────────────────────┐
│                    PERPETUAL SWAP MECHANICS                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Spot Price: $45,000     Perp Price: $45,100                │
│                                                              │
│  Premium = (Perp - Spot) / Spot = +0.22%                    │
│                                                              │
│  Perp > Spot → Positive Funding → Longs pay Shorts          │
│  Perp < Spot → Negative Funding → Shorts pay Longs          │
│                                                              │
│  This mechanism keeps Perp price anchored to Spot           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Linear vs Inverse Contracts**:

| Aspect | Linear (USDT-margined) | Inverse (Coin-margined) |
|--------|------------------------|-------------------------|
| Margin | USDT/USDC | BTC/ETH (the underlying) |
| PnL Currency | USDT | BTC |
| Contract Value | Fixed USD value | Fixed coin quantity |
| Example | BTCUSDT-PERP | BTCUSD-PERP |
| Calculation | Simple (linear) | Non-linear (1/price) |

**Inverse Contract PnL Formula**:
```
PnL (in BTC) = Contracts × (1/Entry - 1/Exit)

Example: Long 10,000 contracts
Entry: $40,000, Exit: $50,000
PnL = 10,000 × (1/40,000 - 1/50,000)
PnL = 10,000 × (0.000025 - 0.00002)
PnL = 0.05 BTC profit
```

### 1.3 Futures Contracts

**Definition**: Agreement to buy/sell an asset at a predetermined price on a specific future date.

**Types**:
| Type | Expiry | Example | Settlement |
|------|--------|---------|------------|
| Weekly | Every Friday | BTC-250124 | Cash or Physical |
| Bi-weekly | Every 2 weeks | BTC-250131 | Cash |
| Quarterly | Mar/Jun/Sep/Dec | BTC-250328 | Cash |
| Bi-quarterly | 6 months out | BTC-250627 | Cash |

**Basis & Contango/Backwardation**:
```
Basis = Futures Price - Spot Price

Contango (Normal):     Futures > Spot  (positive basis)
                       Market expects price to rise

Backwardation:         Futures < Spot  (negative basis)
                       Market expects price to fall / high demand for hedging

Annualized Basis = (Basis / Spot) × (365 / Days to Expiry) × 100%
```

### 1.4 Options

**Definition**: Contract giving the right (not obligation) to buy/sell at a strike price.

| Term | Definition |
|------|------------|
| Call | Right to BUY at strike price |
| Put | Right to SELL at strike price |
| Strike | Pre-agreed exercise price |
| Premium | Price paid for the option |
| Expiry | When option expires |
| ITM | In-the-money (profitable to exercise) |
| OTM | Out-of-the-money (not profitable) |
| ATM | At-the-money (strike ≈ current price) |
| IV | Implied Volatility |
| Greeks | Delta, Gamma, Theta, Vega, Rho |

---

## 2. Derivatives Mechanics

### 2.1 Funding Rate

**Definition**: Periodic payment between long and short position holders to keep perp price aligned with spot.

**Funding Rate Components**:
```
Funding Rate = Interest Rate + Premium/Discount

Where:
- Interest Rate: Usually 0.01% (fixed base rate)
- Premium = (Perp Mark Price - Spot Index Price) / Spot Index Price
```

**Funding Payment**:
```
Payment = Position Value × Funding Rate

Example:
- Position: 10 BTC long at $50,000 = $500,000
- Funding Rate: +0.01%
- Payment: $500,000 × 0.0001 = $50 (longs PAY shorts)
```

**Funding Intervals by Exchange**:
| Exchange | Interval | Settlement Times (UTC) |
|----------|----------|------------------------|
| Binance | 8 hours | 00:00, 08:00, 16:00 |
| Bybit | 8 hours | 00:00, 08:00, 16:00 |
| OKX | 8 hours | 00:00, 08:00, 16:00 |
| dYdX | 1 hour | Every hour |
| Hyperliquid | 1 hour | Every hour |

**Predicted Funding Rate**:
```
Current Funding: Rate for the current interval (already locked)
Predicted Funding: Estimated rate for next interval (changes real-time)

Predicted Rate = Current Premium × Time-Weight Factor
```

**Data Schema for Funding Rates**:
```sql
CREATE TABLE funding_rates (
    exchange LowCardinality(String),
    symbol String,
    funding_time DateTime64(3),
    funding_rate Decimal64(8),        -- e.g., 0.00010000 = 0.01%
    funding_rate_daily Decimal64(8),   -- Annualized: rate × 3 × 365
    mark_price Decimal64(8),
    index_price Decimal64(8),
    next_funding_time DateTime64(3),
    predicted_rate Decimal64(8)
)
ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(funding_time)
ORDER BY (exchange, symbol, funding_time);
```

### 2.2 Open Interest (OI)

**Definition**: Total number of outstanding derivative contracts that have not been settled.

```
Open Interest Logic:
- Trader A opens LONG  + Trader B opens SHORT  → OI increases by 1
- Trader A closes LONG + Trader B closes SHORT → OI decreases by 1
- Trader A closes LONG + Trader C opens LONG   → OI unchanged (transfer)
```

**OI Interpretation**:
| Price Movement | OI Change | Interpretation |
|----------------|-----------|----------------|
| Price ↑ | OI ↑ | New longs entering (bullish) |
| Price ↑ | OI ↓ | Shorts covering (less bullish) |
| Price ↓ | OI ↑ | New shorts entering (bearish) |
| Price ↓ | OI ↓ | Longs liquidating (less bearish) |

**OI Data Schema**:
```sql
CREATE TABLE open_interest (
    exchange LowCardinality(String),
    symbol String,
    timestamp DateTime64(3),
    open_interest Decimal128(18),        -- In contracts
    open_interest_value Decimal64(2),    -- In USD
    open_interest_change_1h Decimal64(4),
    open_interest_change_24h Decimal64(4)
)
ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (exchange, symbol, timestamp);
```

### 2.3 Long/Short Ratio

**Definition**: Ratio of traders with long positions vs short positions.

**Types of L/S Ratios**:
| Metric | What it Measures |
|--------|------------------|
| Account L/S | Number of accounts long vs short |
| Position L/S | Total position value long vs short |
| Top Trader L/S | Positions of top traders (whales) |
| Taker Buy/Sell | Market order volume buy vs sell |

**Interpretation**:
```
L/S Ratio = 2.0 → 2 longs for every 1 short
L/S Ratio = 0.5 → 1 long for every 2 shorts

Contrarian Signal:
- Extreme high L/S (>2.5) → Too many longs, potential dump
- Extreme low L/S (<0.5)  → Too many shorts, potential squeeze
```

**Data Schema**:
```sql
CREATE TABLE long_short_ratio (
    exchange LowCardinality(String),
    symbol String,
    timestamp DateTime64(3),
    long_account Decimal64(4),      -- e.g., 0.55 = 55% long
    short_account Decimal64(4),     -- e.g., 0.45 = 45% short
    long_short_ratio Decimal64(4),  -- e.g., 1.22
    top_trader_long_short_ratio Decimal64(4),
    taker_buy_sell_ratio Decimal64(4)
)
ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (exchange, symbol, timestamp);
```

### 2.4 Liquidations

**Definition**: Forced closure of a position when margin is insufficient to maintain the position.

**Liquidation Mechanics**:
```
┌─────────────────────────────────────────────────────────────┐
│                    LIQUIDATION PROCESS                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Position: 10 BTC Long at $50,000 with 10x leverage         │
│  Margin: $50,000 (10% of position value)                    │
│                                                              │
│  Maintenance Margin: ~0.5% of position = $2,500             │
│                                                              │
│  Liquidation Price ≈ Entry × (1 - 1/Leverage + Maint%)      │
│  Liquidation Price ≈ $50,000 × (1 - 0.1 + 0.005)            │
│  Liquidation Price ≈ $45,250                                │
│                                                              │
│  When BTC hits $45,250:                                     │
│  1. Position forcibly closed                                 │
│  2. Insurance fund covers any deficit                        │
│  3. Liquidation fee charged (~0.5%)                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Liquidation Cascade**:
```
Price drops → Liquidations triggered → More sell pressure →
Price drops further → More liquidations → Cascade effect

This is why liquidation data is critical for:
- Risk management
- Identifying support/resistance levels
- Detecting market manipulation
```

**Data Schema**:
```sql
CREATE TABLE liquidations (
    exchange LowCardinality(String),
    symbol String,
    timestamp DateTime64(3),
    side Enum8('long' = 1, 'short' = 2),
    quantity Decimal64(8),
    price Decimal64(8),
    value_usd Decimal64(2),
    order_id String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (exchange, symbol, timestamp);

-- Aggregated liquidations (for dashboards)
CREATE TABLE liquidations_aggregated (
    exchange LowCardinality(String),
    symbol String,
    timestamp DateTime64(3),  -- Bucketed (1m, 5m, 1h)
    timeframe LowCardinality(String),
    long_liquidations Decimal64(2),
    short_liquidations Decimal64(2),
    total_liquidations Decimal64(2),
    largest_single_liquidation Decimal64(2)
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (exchange, symbol, timeframe, timestamp);
```

### 2.5 Insurance Fund

**Definition**: Reserve fund maintained by exchanges to cover losses from bankrupt positions.

```
Insurance Fund Flow:
1. Profitable liquidations → Surplus goes to insurance fund
2. Unprofitable liquidations → Deficit covered by insurance fund
3. If fund depleted → Auto-deleveraging (ADL) kicks in

ADL: Profitable positions forcibly reduced to cover losses
```

**Why Track Insurance Fund**:
- Fund growth = Exchange health indicator
- Sudden drops = Large liquidation events
- Depletion risk = Potential ADL events

---

## 3. Key Metrics & Indicators

### 3.1 Mark Price vs Last Price vs Index Price

| Price Type | Definition | Usage |
|------------|------------|-------|
| **Last Price** | Most recent trade price | Display, charts |
| **Mark Price** | Fair value (prevents manipulation) | PnL, liquidations |
| **Index Price** | Weighted average from spot exchanges | Funding rate calc |

**Mark Price Calculation**:
```
Mark Price = Index Price × (1 + Funding Basis)

Or (simplified):
Mark Price = Median(Last Price, Best Bid, Best Ask)
             Clamped to ±0.5% of Index Price
```

### 3.2 Order Book Depth & Liquidity

**Level 2 (L2) Data**: Aggregated order book by price level
```
Bid Side          |  Ask Side
Price    Qty      |  Price    Qty
45,000   10.5 BTC |  45,001   8.2 BTC
44,999   5.2 BTC  |  45,002   12.1 BTC
44,998   25.0 BTC |  45,003   3.5 BTC
```

**Level 3 (L3) Data**: Individual orders (rare in crypto)
```
Order ID | Side | Price  | Qty
ABC123   | Bid  | 45,000 | 2.5 BTC
DEF456   | Bid  | 45,000 | 8.0 BTC  ← Same price, different orders
```

**Key Metrics**:
| Metric | Formula | Meaning |
|--------|---------|---------|
| Spread | Ask - Bid | Liquidity indicator |
| Mid Price | (Bid + Ask) / 2 | Fair price estimate |
| Depth | Sum of qty within X% | Liquidity at price levels |
| Imbalance | (Bid Qty - Ask Qty) / Total | Buy/sell pressure |

**Order Book Schema**:
```sql
CREATE TABLE order_book_snapshots (
    exchange LowCardinality(String),
    symbol String,
    timestamp DateTime64(3),
    side Enum8('bid' = 1, 'ask' = 2),
    price Decimal64(8),
    quantity Decimal64(8),
    level UInt16  -- 0 = best bid/ask, 1 = second best, etc.
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (exchange, symbol, timestamp, side, level)
TTL timestamp + INTERVAL 7 DAY;  -- Order book data expires quickly
```

### 3.3 Volume Metrics

| Metric | Definition |
|--------|------------|
| Volume | Total traded quantity in period |
| Turnover | Total traded value (qty × price) |
| Taker Volume | Market orders (aggressive) |
| Maker Volume | Limit orders (passive) |
| Buy Volume | Taker buys (lifting asks) |
| Sell Volume | Taker sells (hitting bids) |

**CVD (Cumulative Volume Delta)**:
```
CVD = Cumulative(Buy Volume - Sell Volume)

Rising CVD + Rising Price = Strong buying (bullish)
Falling CVD + Rising Price = Weak rally (bearish divergence)
Rising CVD + Falling Price = Accumulation (bullish divergence)
```

### 3.4 Realized vs Unrealized PnL

```
Unrealized PnL = (Current Price - Entry Price) × Position Size
                 Position is still open

Realized PnL = (Exit Price - Entry Price) × Position Size
               Position has been closed
```

### 3.5 Basis & Carry Trade

```
Basis = Futures Price - Spot Price

Annualized Basis (APY) = (Basis / Spot) × (365 / Days to Expiry) × 100

Carry Trade Strategy:
1. Buy spot BTC
2. Sell futures BTC
3. Collect basis as yield
4. At expiry: futures converge to spot → profit = basis
```

---

## 4. Exchange Ecosystem

### 4.1 Major Centralized Exchanges (CEX)

| Exchange | Strengths | API Quirks |
|----------|-----------|------------|
| **Binance** | Highest liquidity, most pairs | Weight limits (1200/min), IP bans aggressive |
| **Bybit** | Fast API, good derivatives | Cursor pagination, rate limits per endpoint |
| **OKX** | Good options, portfolio margin | Complex authentication, frequent maintenance |
| **Bitget** | Copy trading data | Limited historical data |
| **Kraken** | Fiat on/off ramp | Slow API, nonce issues |
| **Coinbase** | US regulated, institutional | REST-heavy, limited derivatives |

### 4.2 Decentralized Exchanges (DEX)

| Protocol | Type | Data Source |
|----------|------|-------------|
| **Hyperliquid** | Perps DEX | WebSocket API, on-chain |
| **dYdX** | Perps DEX | REST/WS API, indexer |
| **GMX** | Perps DEX | Arbitrum/Avalanche RPC |
| **Uniswap** | Spot DEX | Ethereum events, The Graph |
| **Jupiter** | Solana aggregator | Solana RPC, API |
| **Raydium** | Solana AMM | Solana RPC |

### 4.3 Data Aggregators

| Provider | Data Offered | Access Method |
|----------|--------------|---------------|
| **Coinglass** | OI, Funding, Liquidations | API (paid) |
| **DefiLlama** | TVL, yields, chains | Free API |
| **CryptoQuant** | On-chain + exchange flows | API (paid) |
| **Glassnode** | On-chain metrics | API (paid) |
| **Kaiko** | Institutional market data | API (enterprise) |
| **CoinGecko** | Prices, metadata | Free API (limited) |
| **CoinMarketCap** | Prices, metadata | Free API (limited) |

### 4.4 Exchange API Rate Limits

**Binance**:
```
- Request Weight: 1200/minute per IP
- Orders: 10/second, 100,000/day
- WebSocket: 5 messages/second per connection
- Connections: 300 per 5 minutes

Weight Examples:
- GET /api/v3/ticker/price: 1 weight
- GET /api/v3/depth?limit=1000: 10 weight
- GET /api/v3/klines: 1 weight
```

**Bybit**:
```
- REST: 120 requests/second (varies by endpoint)
- WebSocket: 500 subscriptions per connection
- Order: 100/second per account

Rate Limit Headers:
- X-Bapi-Limit: Total limit
- X-Bapi-Limit-Status: Remaining
- X-Bapi-Limit-Reset-Timestamp: Reset time
```

**Handling Rate Limits**:
```python
# Exponential backoff pattern
def fetch_with_retry(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url)

        if response.status_code == 429:  # Rate limited
            retry_after = int(response.headers.get('Retry-After', 2 ** attempt))
            time.sleep(retry_after)
            continue

        if response.status_code == 418:  # IP banned (Binance)
            # Switch IP or wait extended period
            time.sleep(300)
            continue

        return response

    raise Exception("Max retries exceeded")
```

---

## 5. Data Sourcing & Aggregation

### 5.1 WebSocket vs REST

| Aspect | WebSocket | REST |
|--------|-----------|------|
| Latency | ~1-10ms | ~50-200ms |
| Data Type | Real-time streams | Snapshots, historical |
| Connection | Persistent | Request/response |
| Use Case | Live trades, order book | Backfill, metadata |

**When to Use Each**:
```
WebSocket:
- Real-time trade feed
- Order book updates
- Funding rate changes
- Liquidation alerts

REST:
- Historical OHLCV backfill
- Contract specifications
- Account balances
- Order placement
```

### 5.2 WebSocket Connection Patterns

**Connection Lifecycle**:
```
┌─────────────────────────────────────────────────────────────┐
│                  WEBSOCKET LIFECYCLE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. CONNECT                                                  │
│     └─► Establish TCP connection                            │
│     └─► WebSocket handshake                                 │
│     └─► Authentication (if required)                        │
│                                                              │
│  2. SUBSCRIBE                                                │
│     └─► Send subscription messages                          │
│     └─► Confirm subscription success                        │
│     └─► Track subscribed channels                           │
│                                                              │
│  3. RECEIVE & PROCESS                                        │
│     └─► Parse incoming messages                             │
│     └─► Handle different message types                      │
│     └─► Push to processing queue                            │
│                                                              │
│  4. HEARTBEAT                                                │
│     └─► Send ping (every 15-30s)                           │
│     └─► Expect pong response                                │
│     └─► Reconnect if no pong                                │
│                                                              │
│  5. RECONNECT (on failure)                                   │
│     └─► Exponential backoff                                 │
│     └─► Re-authenticate                                     │
│     └─► Re-subscribe all channels                           │
│     └─► Detect and fill data gaps                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Self-Healing Mechanisms

**Common Failure Modes**:
| Failure | Detection | Recovery |
|---------|-----------|----------|
| Connection drop | No pong, socket closed | Reconnect with backoff |
| Rate limit (429) | HTTP status code | Pause, exponential backoff |
| IP ban (418) | HTTP status code | Rotate IP, wait 5min+ |
| Data gap | Sequence number jump | REST backfill |
| Stale data | No updates for X seconds | Force reconnect |
| Exchange maintenance | Error message, status page | Wait, retry |

**Gap Detection Pattern**:
```python
class GapDetector:
    def __init__(self):
        self.last_trade_id = {}

    def check_gap(self, symbol, trade_id):
        last_id = self.last_trade_id.get(symbol)

        if last_id and trade_id > last_id + 1:
            # Gap detected!
            gap_start = last_id + 1
            gap_end = trade_id - 1
            self.backfill_trades(symbol, gap_start, gap_end)

        self.last_trade_id[symbol] = trade_id

    def backfill_trades(self, symbol, start_id, end_id):
        # Use REST API to fetch missing trades
        trades = rest_client.get_trades(
            symbol=symbol,
            from_id=start_id,
            limit=1000
        )
        # Process and store backfilled trades
```

### 5.4 Symbol/Ticker Normalization

**The Problem**:
```
Same asset, different symbols across exchanges:

Binance:    BTCUSDT, BTCUSDT-PERP, BTCUSD_PERP
Bybit:      BTCUSDT, BTC-PERP
OKX:        BTC-USDT, BTC-USDT-SWAP, BTC-USD-SWAP
BitMEX:     XBTUSD, XBTUSDT
Hyperliquid: BTC
dYdX:       BTC-USD
```

**Normalization Schema**:
```sql
CREATE TABLE symbol_mappings (
    canonical_symbol String,           -- 'BTC-USDT-PERP'
    exchange LowCardinality(String),   -- 'binance'
    exchange_symbol String,            -- 'BTCUSDT'
    instrument_type Enum8(
        'spot' = 1,
        'perp' = 2,
        'future' = 3,
        'option' = 4
    ),
    base_asset String,                 -- 'BTC'
    quote_asset String,                -- 'USDT'
    settle_asset String,               -- 'USDT' (for inverse: 'BTC')
    contract_type Enum8(
        'linear' = 1,
        'inverse' = 2
    ),
    is_active UInt8,
    updated_at DateTime64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (exchange, exchange_symbol);

-- Lookup: Exchange symbol → Canonical
SELECT canonical_symbol
FROM symbol_mappings
WHERE exchange = 'binance' AND exchange_symbol = 'BTCUSDT';
-- Returns: 'BTC-USDT-SPOT'
```

**Normalization Function**:
```python
def normalize_symbol(exchange: str, exchange_symbol: str) -> dict:
    """
    Convert exchange-specific symbol to canonical format.
    Returns dict with base, quote, instrument type, etc.
    """
    patterns = {
        'binance': {
            r'(\w+)USDT$': lambda m: {
                'base': m.group(1),
                'quote': 'USDT',
                'type': 'spot'
            },
            r'(\w+)USDT-PERP$': lambda m: {
                'base': m.group(1),
                'quote': 'USDT',
                'type': 'perp',
                'contract': 'linear'
            },
            r'(\w+)USD_PERP$': lambda m: {
                'base': m.group(1),
                'quote': 'USD',
                'type': 'perp',
                'contract': 'inverse'
            },
        },
        'bitmex': {
            r'XBT(\w+)$': lambda m: {
                'base': 'BTC',
                'quote': m.group(1) or 'USD',
                'type': 'perp'
            },
        }
    }
    # ... pattern matching logic
```

### 5.5 Multi-Exchange Aggregation

**Price Aggregation Methods**:
| Method | Formula | Use Case |
|--------|---------|----------|
| Simple Average | sum(prices) / n | Equal weight |
| Volume-Weighted | sum(price × vol) / sum(vol) | Fair price |
| Median | middle value | Outlier resistant |
| TWAP | time-weighted average | Reference price |

**Index Price Calculation** (similar to exchange index):
```sql
-- Create aggregated index price
SELECT
    toStartOfMinute(timestamp) AS minute,
    base_asset,
    -- Volume-weighted average price
    sum(price * volume) / sum(volume) AS vwap,
    -- Median price
    median(price) AS median_price,
    -- Data quality
    count(DISTINCT exchange) AS exchange_count,
    sum(volume) AS total_volume
FROM trades_normalized
WHERE timestamp > now() - INTERVAL 1 HOUR
  AND instrument_type = 'spot'
  AND base_asset = 'BTC'
  AND quote_asset = 'USDT'
GROUP BY minute, base_asset
ORDER BY minute;
```

---

## 6. Pipeline Architecture

### 6.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA PIPELINE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐               │
│  │   Binance    │    │    Bybit     │    │     OKX      │    ...        │
│  │  WebSocket   │    │  WebSocket   │    │  WebSocket   │               │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘               │
│         │                   │                   │                        │
│         └───────────────────┼───────────────────┘                        │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      MESSAGE QUEUE (NATS)                        │    │
│  │   Subjects: trades.*, funding.*, liquidations.*, orderbook.*    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                             │                                            │
│         ┌───────────────────┼───────────────────┐                        │
│         ▼                   ▼                   ▼                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐               │
│  │  Normalizer  │    │  Aggregator  │    │   Enricher   │               │
│  │  (symbols)   │    │   (OHLCV)    │    │  (metadata)  │               │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘               │
│         │                   │                   │                        │
│         └───────────────────┼───────────────────┘                        │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        CLICKHOUSE                                │    │
│  │                                                                  │    │
│  │   trades_raw    │   ohlcv_1m/5m/1h   │   funding_rates          │    │
│  │   order_book    │   liquidations     │   positions               │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        REST API                                  │    │
│  │        Charts │ Metrics │ Alerts │ Backtesting                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Connector Architecture

**Single Exchange Connector**:
```
┌─────────────────────────────────────────────────────────────┐
│                   EXCHANGE CONNECTOR                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐       ┌─────────────────┐              │
│  │  WebSocket Mgr  │◄─────►│   REST Client   │              │
│  │  - Trades       │       │  - Backfill     │              │
│  │  - Order Book   │       │  - Metadata     │              │
│  │  - Funding      │       │  - Symbols      │              │
│  └────────┬────────┘       └─────────────────┘              │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────────────────────────────────┐            │
│  │              Message Handler                 │            │
│  │  - Parse exchange format                     │            │
│  │  - Normalize to internal schema              │            │
│  │  - Sequence number tracking                  │            │
│  │  - Gap detection                             │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                      │
│                       ▼                                      │
│  ┌─────────────────────────────────────────────┐            │
│  │              Health Monitor                  │            │
│  │  - Connection state                          │            │
│  │  - Message rate                              │            │
│  │  - Latency tracking                          │            │
│  │  - Auto-reconnect                            │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                      │
│                       ▼                                      │
│  ┌─────────────────────────────────────────────┐            │
│  │            Output (NATS Publish)             │            │
│  └─────────────────────────────────────────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Message Queue Patterns (NATS)

**Subject Hierarchy**:
```
trades.{exchange}.{symbol}
  └─► trades.binance.btc-usdt-spot
  └─► trades.bybit.btc-usdt-perp

funding.{exchange}.{symbol}
  └─► funding.binance.btc-usdt-perp

liquidations.{exchange}.{symbol}
  └─► liquidations.binance.btc-usdt-perp

orderbook.{exchange}.{symbol}.{depth}
  └─► orderbook.binance.btc-usdt-spot.l2
```

**Consumer Groups (Queue Groups)**:
```
# Multiple consumers share load
nats.subscribe("trades.>", queue="trade-processors")

# All consumers receive all messages (fanout)
nats.subscribe("trades.>")  # No queue = fanout
```

---

## 7. Storage Patterns

### 7.1 Time-Series Data Characteristics

**Crypto Market Data Properties**:
| Property | Impact on Storage |
|----------|-------------------|
| 24/7 operation | No "quiet hours", consistent load |
| High frequency | Millions of trades/day |
| Append-only | Rarely update historical data |
| Time-ordered queries | Primary access pattern |
| Multi-dimensional | Filter by exchange, symbol, time |

### 7.2 Precision Handling

**The Challenge**:
```
Different exchanges use different precision:

Binance BTC price:  45123.45000000 (8 decimals)
Bybit BTC price:    45123.50 (2 decimals)
Solana token:       0.00000001234567890123456789 (18+ decimals)
```

**Solution: Use Appropriate Types**:
```sql
-- Price precision by asset class
price_usd Decimal64(8)        -- Major pairs: 8 decimals
price_usd Decimal128(18)      -- DeFi/Solana: 18 decimals

-- Quantity precision
quantity Decimal64(8)         -- Most exchanges
quantity Decimal128(18)       -- Small tokens (wei, lamports)

-- Avoid Float64 for financial data!
-- Float64 has precision issues: 0.1 + 0.2 ≠ 0.3
```

### 7.3 Timestamp Handling

**Exchange Timestamp Formats**:
| Exchange | Format | Example |
|----------|--------|---------|
| Binance | Milliseconds | 1705766400000 |
| Bybit | Milliseconds | 1705766400000 |
| BitMEX | ISO 8601 | 2024-01-20T12:00:00.000Z |
| Solana | Seconds (Unix) | 1705766400 |
| Ethereum | Seconds (block time) | 1705766400 |

**Normalization**:
```sql
-- Always store as DateTime64(3) for millisecond precision
timestamp DateTime64(3)

-- Conversion functions
toDateTime64(1705766400000 / 1000, 3)  -- From milliseconds
toDateTime64(1705766400, 3)             -- From seconds
parseDateTimeBestEffort('2024-01-20T12:00:00Z')  -- From ISO
```

### 7.4 Contract Specification Storage

```sql
CREATE TABLE contract_specs (
    exchange LowCardinality(String),
    symbol String,
    instrument_type Enum8('spot'=1, 'perp'=2, 'future'=3, 'option'=4),

    -- Contract details
    base_asset String,
    quote_asset String,
    settle_asset String,
    contract_type Enum8('linear'=1, 'inverse'=2, 'quanto'=3),

    -- Sizing
    contract_size Decimal64(8),        -- 1 contract = X units
    tick_size Decimal64(8),            -- Minimum price movement
    lot_size Decimal64(8),             -- Minimum order quantity
    min_notional Decimal64(2),         -- Minimum order value (USD)
    max_leverage UInt16,

    -- Margin
    initial_margin_rate Decimal64(4),  -- e.g., 0.10 = 10%
    maintenance_margin_rate Decimal64(4),

    -- Fees
    maker_fee Decimal64(6),
    taker_fee Decimal64(6),

    -- Status
    status Enum8('trading'=1, 'settling'=2, 'delisted'=3),
    listing_date Date,
    expiry_date Nullable(Date),        -- NULL for perps

    updated_at DateTime64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (exchange, symbol);

-- Example data
INSERT INTO contract_specs VALUES (
    'binance', 'BTCUSDT', 'perp',
    'BTC', 'USDT', 'USDT', 'linear',
    1.0,        -- 1 contract = 1 BTC
    0.10,       -- Tick size: $0.10
    0.001,      -- Lot size: 0.001 BTC
    5.0,        -- Min notional: $5
    125,        -- Max leverage: 125x
    0.008,      -- Initial margin: 0.8%
    0.004,      -- Maintenance margin: 0.4%
    0.0002,     -- Maker fee: 0.02%
    0.0004,     -- Taker fee: 0.04%
    'trading',
    '2019-09-13', NULL,
    now()
);
```

---

## 8. Data Quality & Anomaly Handling

### 8.1 Common Data Anomalies

| Anomaly | Description | Impact |
|---------|-------------|--------|
| Flash crash wick | Single trade at extreme price | Skews OHLC, triggers false signals |
| Bad print | Erroneous trade (fat finger) | False liquidations in backtest |
| Duplicate trades | Same trade ID received twice | Inflated volume |
| Out-of-order | Late arriving trades | Incorrect sequence |
| Missing data | Gap in feed | Incomplete candles |
| Stale quotes | Order book not updated | Wrong spread calculation |

### 8.2 Flash Crash Detection

**Detection Methods**:
```sql
-- Method 1: Price deviation from recent average
SELECT *
FROM trades
WHERE abs(price - avg_price_5m) / avg_price_5m > 0.05  -- >5% deviation

-- Method 2: Price vs mark price
SELECT *
FROM trades t
JOIN mark_prices m ON t.symbol = m.symbol
  AND t.timestamp BETWEEN m.timestamp - INTERVAL 1 SECOND
                      AND m.timestamp + INTERVAL 1 SECOND
WHERE abs(t.price - m.mark_price) / m.mark_price > 0.03  -- >3% from mark

-- Method 3: Isolation (single trade spike)
SELECT *
FROM (
    SELECT
        *,
        lag(price) OVER (PARTITION BY symbol ORDER BY timestamp) AS prev_price,
        lead(price) OVER (PARTITION BY symbol ORDER BY timestamp) AS next_price
    FROM trades
)
WHERE abs(price - prev_price) / prev_price > 0.02
  AND abs(price - next_price) / next_price > 0.02
  -- Price spiked and immediately returned
```

**Filtering Strategy**:
```sql
-- Create clean trades view
CREATE VIEW trades_clean AS
SELECT *
FROM trades t
WHERE NOT EXISTS (
    SELECT 1 FROM anomalies a
    WHERE a.trade_id = t.trade_id
)
AND price BETWEEN
    (SELECT percentile(0.001)(price) FROM trades WHERE symbol = t.symbol AND timestamp > t.timestamp - INTERVAL 1 HOUR)
    AND
    (SELECT percentile(0.999)(price) FROM trades WHERE symbol = t.symbol AND timestamp > t.timestamp - INTERVAL 1 HOUR);
```

### 8.3 Deduplication

**Trade Deduplication**:
```sql
-- Use ReplacingMergeTree with trade_id as version
CREATE TABLE trades (
    exchange LowCardinality(String),
    symbol String,
    trade_id String,
    timestamp DateTime64(3),
    price Decimal64(8),
    quantity Decimal64(8),
    side Enum8('buy'=1, 'sell'=2)
)
ENGINE = ReplacingMergeTree()
ORDER BY (exchange, symbol, trade_id);

-- Duplicates automatically removed during merge
```

### 8.4 Data Validation Rules

```sql
-- Validation checks on insert
CREATE MATERIALIZED VIEW trades_validation TO trades_errors AS
SELECT
    *,
    multiIf(
        price <= 0, 'invalid_price',
        quantity <= 0, 'invalid_quantity',
        timestamp > now() + INTERVAL 1 MINUTE, 'future_timestamp',
        timestamp < now() - INTERVAL 1 DAY, 'stale_timestamp',
        'valid'
    ) AS validation_status
FROM trades_raw
WHERE validation_status != 'valid';
```

---

## 9. Reference Data Management

### 9.1 Symbol Master

```sql
CREATE TABLE symbol_master (
    symbol_id UInt64,
    canonical_symbol String,           -- 'BTC-USDT-PERP'

    -- Asset info
    base_asset String,
    base_asset_name String,            -- 'Bitcoin'
    quote_asset String,

    -- Classification
    asset_class Enum8('crypto'=1, 'stablecoin'=2, 'wrapped'=3),
    sector LowCardinality(String),     -- 'L1', 'DeFi', 'Meme'

    -- Identifiers
    coingecko_id Nullable(String),
    coinmarketcap_id Nullable(UInt32),

    -- Metadata
    logo_url Nullable(String),
    website Nullable(String),

    is_active UInt8,
    created_at DateTime64(3),
    updated_at DateTime64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY symbol_id;
```

### 9.2 Exchange Status Tracking

```sql
CREATE TABLE exchange_status (
    exchange LowCardinality(String),
    timestamp DateTime64(3),

    -- Connection status
    ws_connected UInt8,
    rest_available UInt8,

    -- Health metrics
    message_rate Float32,              -- Messages per second
    latency_ms Float32,                -- Average latency
    error_rate Float32,                -- Errors per minute

    -- Maintenance
    is_maintenance UInt8,
    maintenance_message String,

    updated_at DateTime64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (exchange, timestamp);
```

### 9.3 Data Freshness Monitoring

```sql
-- Check data freshness by source
SELECT
    exchange,
    symbol,
    max(timestamp) AS last_trade,
    dateDiff('second', max(timestamp), now()) AS seconds_stale,
    CASE
        WHEN dateDiff('second', max(timestamp), now()) < 5 THEN 'fresh'
        WHEN dateDiff('second', max(timestamp), now()) < 60 THEN 'delayed'
        ELSE 'stale'
    END AS status
FROM trades
WHERE timestamp > now() - INTERVAL 1 HOUR
GROUP BY exchange, symbol
HAVING seconds_stale > 10
ORDER BY seconds_stale DESC;
```

---

## 10. Complete Glossary

### Trading Terms

| Term | Definition |
|------|------------|
| **ADL** | Auto-Deleveraging - forced position reduction when insurance fund depleted |
| **Ask** | Lowest price a seller is willing to accept |
| **ATH** | All-Time High |
| **ATL** | All-Time Low |
| **Backwardation** | Futures price < Spot price |
| **Basis** | Difference between futures and spot price |
| **Bid** | Highest price a buyer is willing to pay |
| **Carry** | Profit from holding futures vs spot (funding + basis) |
| **Contango** | Futures price > Spot price |
| **CVD** | Cumulative Volume Delta (buy volume - sell volume) |
| **Delta** | Rate of change of option price vs underlying |
| **Depth** | Total order volume at price levels |
| **Fill** | Executed portion of an order |
| **Funding Rate** | Periodic payment between longs and shorts |
| **Greeks** | Option sensitivity measures (Delta, Gamma, Theta, Vega) |
| **Imbalance** | Difference between buy and sell pressure |
| **Index Price** | Weighted average spot price from multiple exchanges |
| **Inverse Contract** | Contract margined in base asset (e.g., BTC) |
| **IV** | Implied Volatility |
| **Leverage** | Position size / Margin |
| **Linear Contract** | Contract margined in quote asset (e.g., USDT) |
| **Liquidation** | Forced position closure due to insufficient margin |
| **Long** | Bet that price will increase |
| **Maker** | Provides liquidity (limit order) |
| **Margin** | Collateral required to open position |
| **Mark Price** | Fair value price used for liquidations |
| **OI** | Open Interest - total outstanding contracts |
| **Perp** | Perpetual swap - no expiry derivative |
| **PnL** | Profit and Loss |
| **Premium** | Options: price paid for contract. Futures: basis |
| **RVOL** | Realized Volatility |
| **Short** | Bet that price will decrease |
| **Slippage** | Difference between expected and actual execution price |
| **Spread** | Ask price - Bid price |
| **Strike** | Pre-agreed price in options contract |
| **Taker** | Removes liquidity (market order) |
| **TWAP** | Time-Weighted Average Price |
| **VWAP** | Volume-Weighted Average Price |

### Technical Terms

| Term | Definition |
|------|------------|
| **Backfill** | Fetching historical data to fill gaps |
| **Backpressure** | Queue buildup when consumer is slower than producer |
| **CDC** | Change Data Capture |
| **Cursor** | Pagination pointer for REST APIs |
| **Granule** | ClickHouse data block (~8192 rows) |
| **Heartbeat** | Periodic ping to maintain connection |
| **Idempotent** | Operation that produces same result if repeated |
| **L2 Data** | Level 2 - aggregated order book by price |
| **L3 Data** | Level 3 - individual orders |
| **Latency** | Time delay in data transmission |
| **MV** | Materialized View |
| **Nonce** | Number used once (for API authentication) |
| **Part** | ClickHouse data file segment |
| **Projection** | Pre-computed alternate sort order in ClickHouse |
| **Rate Limit** | Maximum requests allowed per time period |
| **Reconnect** | Re-establishing dropped connection |
| **Sequence Number** | Ordered identifier for messages |
| **Snapshot** | Point-in-time state (e.g., order book snapshot) |
| **TTL** | Time-To-Live - automatic data expiration |
| **WebSocket** | Full-duplex communication protocol |

### Exchange-Specific Terms

| Term | Exchange | Meaning |
|------|----------|---------|
| **Weight** | Binance | API request cost unit |
| **UID** | Bybit | User identifier |
| **Instrument ID** | OKX | Symbol identifier |
| **XBT** | BitMEX | Bitcoin symbol |
| **Quanto** | BitMEX | Contract with non-linear PnL |

---

## Summary

This document provides the theoretical foundation for crypto data engineering:

1. **Market Structure**: Spot, Perps, Futures, Options
2. **Derivatives Mechanics**: Funding, OI, Liquidations, Insurance Fund
3. **Key Metrics**: Mark/Index/Last Price, CVD, Basis
4. **Exchange Ecosystem**: CEX vs DEX, API patterns, Rate limits
5. **Data Sourcing**: WebSocket vs REST, Self-healing, Normalization
6. **Pipeline Architecture**: Connectors, Message queues, Processing
7. **Storage Patterns**: Precision, Timestamps, Contract specs
8. **Data Quality**: Anomaly detection, Deduplication, Validation
9. **Reference Data**: Symbol master, Exchange status, Freshness

This knowledge is essential for building reliable, production-grade crypto data infrastructure.
