# Building Real-Time Solana DEX Analytics with ClickHouse

## Part 2: Trade Enrichment, OHLCV & MergeTree Deep Dive

This part covers the core analytics: transforming raw Solana swaps into enriched trades and OHLCV candles, with deep dives into ClickHouse optimizations that enable sub-200ms queries at 100K+ trades/second.

---

## Table of Contents

1. [Theory: Understanding Solana DEX Trades](#theory-understanding-solana-dex-trades)
2. [Trade Enrichment: From Raw Swap to Analytics](#trade-enrichment-from-raw-swap-to-analytics)
3. [Complete Enriched Trade Metrics (45 Fields)](#complete-enriched-trade-metrics-45-fields)
4. [Fee Decomposition: 5 Fee Categories](#fee-decomposition-5-fee-categories)
5. [Theory: OHLCV Candlestick Aggregation](#theory-ohlcv-candlestick-aggregation)
6. [OHLCV Metrics (28 Fields per Candle)](#ohlcv-metrics-28-fields-per-candle)
7. [ClickHouse Deep Dive: MergeTree Engine](#clickhouse-deep-dive-mergetree-engine)
8. [ClickHouse Optimization: ORDER BY Strategy](#clickhouse-optimization-order-by-strategy)
9. [ClickHouse Optimization: Data Skipping Indexes](#clickhouse-optimization-data-skipping-indexes)
10. [ClickHouse Optimization: Tiered Storage & TTL](#clickhouse-optimization-tiered-storage--ttl)
11. [ClickHouse Optimization: Compression Codecs](#clickhouse-optimization-compression-codecs)
12. [Complete DDL: Enriched Trades](#complete-ddl-enriched-trades)
13. [Complete DDL: OHLCV Tiered Tables](#complete-ddl-ohlcv-tiered-tables)
14. [Materialized View Patterns](#materialized-view-patterns)
15. [Query Patterns & Performance](#query-patterns--performance)

---

## Theory: Understanding Solana DEX Trades

### How Solana DEX Swaps Work

Unlike traditional order-book exchanges, Solana DEXes use Automated Market Makers (AMMs):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AMM Pool Mechanics                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Pool: SOL/BONK (Raydium)                                              │
│   ┌─────────────────┐     ┌─────────────────┐                           │
│   │  Reserve A      │     │  Reserve B      │                           │
│   │  10,000 SOL     │ ←→  │  1B BONK        │                           │
│   └─────────────────┘     └─────────────────┘                           │
│                                                                          │
│   Constant Product Formula: x × y = k                                    │
│   10,000 × 1,000,000,000 = 10,000,000,000,000                           │
│                                                                          │
│   Spot Price: 1 SOL = 100,000 BONK (or 1 BONK = 0.00001 SOL)            │
│                                                                          │
│   User Swaps 1 SOL → BONK:                                              │
│   - New SOL reserve: 10,001                                              │
│   - New BONK reserve: k / 10,001 = 999,900,009.99                       │
│   - User receives: 99,990 BONK                                          │
│   - Execution price: 1 SOL = 99,990 BONK (0.01% slippage)               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Concepts for Analytics

| Concept | Definition | Why It Matters |
|---------|------------|----------------|
| **Spot Price** | Price before trade executes | Reference for slippage calculation |
| **Execution Price** | Actual price paid | Real cost to trader |
| **Slippage** | (Execution - Spot) / Spot | Trade quality indicator |
| **Liquidity** | Total reserves × 2 | Determines price impact |
| **Market Cap** | Spot Price × Total Supply | Token valuation |
| **Bonding Curve %** | Progress to graduation (Pump.fun) | Token lifecycle stage |

### Multi-Hop Routes (Jupiter)

Jupiter aggregates across pools for optimal execution:

```
User wants: 1 SOL → BONK

Route A (Direct): SOL → BONK on Raydium
  - Slippage: 2.1% (low liquidity)

Route B (Multi-hop): SOL → USDC → BONK
  - Hop 1: SOL → USDC on Raydium (0.1% slippage)
  - Hop 2: USDC → BONK on Orca (0.3% slippage)
  - Total slippage: 0.4% ✓ Better!

Each hop = separate trade event in our system
```

---

## Trade Enrichment: From Raw Swap to Analytics

### What Raw Blockchain Data Looks Like

```
Raw Swap Event:
├── signature: "5K7p9..."
├── source_mint: "So111111...2" (Wrapped SOL)
├── destination_mint: "DezXAZ...263" (BONK)
├── amount_in: 1000000000 (1 SOL in lamports)
├── amount_out: 99990000000000 (99.99B in raw decimals)
├── execution_price_a_in_b: 99990000000
└── spot_price_a_in_b: 100000000000
```

**Problems:**
- No token names/symbols
- Raw amounts (not decimal-adjusted)
- No USD values
- No market metrics
- No trade direction classification

### What Enriched Data Looks Like

```
Enriched Trade:
├── signature: "5K7p9..."
├── token_mint: "DezXAZ...263"
├── token_name: "Bonk"
├── token_symbol: "BONK"
├── trade_direction: "Buy"
├── volume_sol: 1.0
├── volume_usd: 150.0
├── token_amount: 99,990,000,000
├── spot_price_sol: 0.00000001
├── spot_price_usd: 0.0000015
├── execution_price_sol: 0.000000010001
├── market_cap_sol: 93,450
├── market_cap_usd: 14,017,500
├── liquidity_sol: 10,000
├── liquidity_usd: 1,500,000
└── slippage_pct: 0.01%
```

---

## Complete Enriched Trade Metrics (45 Fields)

### Category 1: Transaction Identity (6 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `signature` | String | Unique 88-char base58 transaction signature | `"5K7p9Qk..."` |
| `slot` | UInt64 | Solana slot number (monotonic block height) | `245123456` |
| `transaction_index` | UInt64 | Position within block | `152` |
| `instruction_index` | UInt8 | Instruction within transaction | `2` |
| `trader` | String | Wallet public key (44 chars) | `"7xKXtg2..."` |
| `market` | String | Pool/AMM address | `"58oQChx..."` |

### Category 2: Token Information (5 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `dex` | LowCardinality(String) | DEX/protocol name | `"raydium"`, `"pump"`, `"orca"` |
| `anchor_mint` | String | Anchor currency (SOL/USDC/USDT) | `"So111..."` |
| `token_mint` | String | Token being traded | `"DezXAZ..."` |
| `token_name` | String | Token name from metadata | `"Bonk"` |
| `token_symbol` | String | Token symbol | `"BONK"` |

### Category 3: Trade Details (5 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `trade_direction` | UInt8 | 0=Buy (acquiring token), 1=Sell (disposing) | `0` |
| `anchor_amount` | Decimal128(18) | Amount of anchor (SOL/USDC) | `1.5` |
| `token_amount` | Decimal128(18) | Amount of tokens | `150000000.0` |
| `volume_sol` | Decimal128(18) | Trade volume in SOL | `1.5` |
| `volume_usd` | Decimal128(18) | Trade volume in USD | `225.0` |

### Category 4: Pricing (4 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `spot_price_sol` | Decimal128(18) | Pre-trade price in SOL | `0.00000001` |
| `spot_price_usd` | Decimal128(18) | Pre-trade price in USD | `0.0000015` |
| `execution_price_sol` | Decimal128(18) | Actual execution price SOL | `0.000000010005` |
| `execution_price_usd` | Decimal128(18) | Actual execution price USD | `0.00000150075` |

### Category 5: Market Metrics (5 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `market_cap_sol` | Decimal128(18) | Market cap at trade time (SOL) | `93450.5` |
| `market_cap_usd` | Decimal128(18) | Market cap at trade time (USD) | `14017575.0` |
| `liquidity_sol` | Decimal128(18) | Pool liquidity (SOL) | `10000.0` |
| `liquidity_usd` | Decimal128(18) | Pool liquidity (USD) | `1500000.0` |
| `token_supply` | Decimal128(18) | Total token supply | `9345000000000.0` |

### Category 6: Multi-Hop Metadata (3 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `hops` | UInt8 | Total hops in route (1=direct) | `3` |
| `hop_index` | UInt8 | This hop's position (1-indexed) | `1` |
| `bonding_curve_pct` | Nullable(Decimal128(18)) | Pump.fun curve % (null if graduated) | `85.5` |

### Category 7: Fee Tracking (10 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `transaction_fee_sol` | Decimal128(18) | Base transaction fee | `0.000005` |
| `transaction_fee_usd` | Decimal128(18) | Transaction fee in USD | `0.00075` |
| `priority_fee_sol` | Decimal128(18) | Priority fee (compute units) | `0.0001` |
| `priority_fee_usd` | Decimal128(18) | Priority fee in USD | `0.015` |
| `validator_tip_sol` | Decimal128(18) | MEV tip (Jito) | `0.001` |
| `validator_tip_usd` | Decimal128(18) | MEV tip in USD | `0.15` |
| `dex_fee_sol` | Decimal128(18) | DEX/AMM fee | `0.015` |
| `dex_fee_usd` | Decimal128(18) | DEX fee in USD | `2.25` |
| `platform_fee_sol` | Decimal128(18) | Trading bot fee | `0.0075` |
| `platform_fee_usd` | Decimal128(18) | Platform fee in USD | `1.125` |

### Category 8: Timestamps (3 fields)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `block_timestamp` | DateTime64(9) | Block consensus time (nanoseconds) | `2024-01-15 10:30:45.123456789` |
| `server_timestamp` | DateTime64(9) | Indexer receipt time | `2024-01-15 10:30:45.234567890` |
| `ingested_at` | DateTime64(3) | ClickHouse ingestion time | `2024-01-15 10:30:45.500` |

### Category 9: Click Platform (4 fields, internal)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `click_user_id` | Nullable(String) | Click user UUID | `"uuid-xxxx"` |
| `click_order_id` | Nullable(String) | Associated order | `"uuid-yyyy"` |
| `click_order_type` | Nullable(String) | Order type | `"limit"`, `"market"` |
| `click_points_earned` | Nullable(Decimal128(18)) | Points from trade | `15.5` |

---

## Fee Decomposition: 5 Fee Categories

### Understanding Solana Transaction Costs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TOTAL COST OF A TRADE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. BASE TRANSACTION FEE                                                │
│     └─ Fixed: 5,000 lamports (0.000005 SOL)                             │
│     └─ Goes to: Validators (burned + leader reward)                     │
│     └─ When: Every transaction                                          │
│                                                                          │
│  2. PRIORITY FEE (Compute Budget)                                       │
│     └─ Variable: 0 to millions of lamports                              │
│     └─ Set via: ComputeBudget.SetComputeUnitPrice                       │
│     └─ Goes to: Block leader (validator)                                │
│     └─ When: Congested periods, MEV opportunities                       │
│                                                                          │
│  3. VALIDATOR TIP (MEV/Jito)                                            │
│     └─ Variable: Direct SOL transfer                                    │
│     └─ Goes to: Jito/Bloxroute validators (72 known accounts)           │
│     └─ When: Need guaranteed fast inclusion                             │
│     └─ Detection: SOL transfers to MEV tip accounts                     │
│                                                                          │
│  4. DEX FEE                                                             │
│     └─ Variable by protocol:                                            │
│        - Pump.fun: 1%                                                   │
│        - Raydium V4: 0.25%                                              │
│        - Orca Whirlpool: 0.1-1% (pool-specific)                        │
│        - Meteora: 0.3%                                                  │
│     └─ Goes to: LPs + Protocol treasury                                 │
│     └─ When: Every swap (per-hop in multi-hop)                          │
│                                                                          │
│  5. PLATFORM FEE (Trading Bots)                                         │
│     └─ Variable: 0.5-1% typically                                       │
│     └─ Goes to: Bot developers (51 known wallets)                       │
│     └─ Platforms: Axiom, Trojan, BullX, Bloom, BONKbot,                 │
│                   Photon, GMGN, Maestro, PepeBoost...                   │
│     └─ Detection: SOL transfers to known platform wallets               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Fee Accumulation Rules for Multi-Hop

| Fee Type | Per-Transaction | Per-Hop | Count On |
|----------|-----------------|---------|----------|
| Transaction Fee | ✓ | | hop_index = 1 |
| Priority Fee | ✓ | | hop_index = 1 |
| Validator Tip | ✓ | | hop_index = 1 |
| DEX Fee | | ✓ | Every hop |
| Platform Fee | ✓ | | hop_index = 1 |

---

## Data Pipeline: Trade Enrichment Flow

### How Trades Flow Through DataStreams

Let's trace exactly how a raw swap becomes an enriched trade:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    TRADE ENRICHMENT PIPELINE                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  INPUT: solana.trades.v2 (Raw Swap from Indexer)                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ {                                                                            ││
│  │   signature: "5K7p9...",                                                     ││
│  │   source_mint: "So111...",        // Wrapped SOL                            ││
│  │   destination_mint: "DezXAZ...",  // Token mint (BONK)                      ││
│  │   amount_in: 1000000000,          // 1 SOL in lamports                      ││
│  │   amount_out: 99990000000000,     // Raw token decimals                     ││
│  │   market: "58oQChx...",           // Pool address                           ││
│  │   trader: "7xKXtg2...",           // Wallet                                 ││
│  │   slot: 245123456,                                                          ││
│  │   dex: "raydium"                                                            ││
│  │ }                                                                            ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                      │                                           │
│                                      ▼                                           │
│  TRANSFORM: enrich_trade (Rustflow)                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ Pipeline: token_prices                                                       ││
│  │ Partition By: token_mint (parallel processing per token)                     ││
│  │                                                                              ││
│  │ Step 1: Load Token Metadata (from state)                                     ││
│  │ ├─ Key: token_mint → "DezXAZ..."                                            ││
│  │ ├─ Lookup: context.state.get::<TokenMetadata>(token_mint)                   ││
│  │ └─ Returns: { name: "Bonk", symbol: "BONK", decimals: 5 }                   ││
│  │                                                                              ││
│  │ Step 2: Load Pool State (from state)                                         ││
│  │ ├─ Key: market → "58oQChx..."                                               ││
│  │ ├─ Lookup: context.state.get::<PoolState>(market)                           ││
│  │ └─ Returns: { reserve_a: 10000 SOL, reserve_b: 1B tokens,                   ││
│  │              liquidity_sol: 20000, spot_price: 0.00001 }                    ││
│  │                                                                              ││
│  │ Step 3: Calculate Derived Fields                                             ││
│  │ ├─ trade_direction: SOL→Token = Buy, Token→SOL = Sell                       ││
│  │ ├─ volume_sol: amount_in / 10^9 = 1.0                                       ││
│  │ ├─ volume_usd: volume_sol × sol_usd_price = 150.0                           ││
│  │ ├─ token_amount: amount_out / 10^decimals = 999,900,000                     ││
│  │ ├─ execution_price: volume_sol / token_amount                               ││
│  │ ├─ market_cap: spot_price × supply                                          ││
│  │ └─ slippage: (execution_price - spot_price) / spot_price                    ││
│  │                                                                              ││
│  │ Step 4: Extract Fees (from transaction instructions)                         ││
│  │ ├─ transaction_fee: Fixed 5000 lamports                                     ││
│  │ ├─ priority_fee: From ComputeBudget instruction                             ││
│  │ ├─ validator_tip: Match transfers to 72 known Jito accounts                 ││
│  │ ├─ dex_fee: Pool fee rate × volume (per-hop)                                ││
│  │ └─ platform_fee: Match transfers to 51 known bot wallets                    ││
│  │                                                                              ││
│  │ Step 5: Emit Enriched Trade                                                  ││
│  │ └─ context.emit("solana.enriched_trades", trade)                            ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                      │                                           │
│                                      ▼                                           │
│  OUTPUT: solana.enriched_trades (45 fields)                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ {                                                                            ││
│  │   signature: "5K7p9...",                                                     ││
│  │   token_mint: "DezXAZ...",                                                   ││
│  │   token_name: "Bonk",          // ← Enriched                                ││
│  │   token_symbol: "BONK",        // ← Enriched                                ││
│  │   trade_direction: 0,          // ← Computed (0=Buy)                        ││
│  │   volume_sol: 1.0,             // ← Normalized                              ││
│  │   volume_usd: 150.0,           // ← USD converted                           ││
│  │   spot_price_sol: 0.00000001,  // ← From pool state                         ││
│  │   market_cap_usd: 14017575,    // ← Calculated                              ││
│  │   transaction_fee_sol: 0.000005,                                             ││
│  │   dex_fee_sol: 0.015,          // ← Extracted                               ││
│  │   platform_fee_sol: 0.01,      // ← Detected (Axiom)                        ││
│  │   ...                                                                        ││
│  │ }                                                                            ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### State Management for Trade Enrichment

The enrichment transform requires state lookups for metadata:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    STATE DEPENDENCIES                                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                         GLOBAL PIPELINE                                      ││
│  │  (Runs first, populates metadata state)                                      ││
│  │                                                                              ││
│  │  solana.tokens ───► store_token ───► TokenMetadata state                    ││
│  │  solana.pools ────► store_pool ────► PoolState state                        ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                      │                                           │
│                    State available for downstream transforms                     │
│                                      │                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                       TOKEN_PRICES PIPELINE                                  ││
│  │  (Reads from state populated by global pipeline)                             ││
│  │                                                                              ││
│  │  solana.trades.v2 ───► enrich_trade ───► solana.enriched_trades             ││
│  │                              │                                               ││
│  │                     Reads: TokenMetadata, PoolState                          ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│  STATE STRUCTURE:                                                               │
│                                                                                  │
│  TokenMetadata {                    PoolState {                                 │
│    mint: String,                      market: String,                           │
│    name: String,                      dex: String,                              │
│    symbol: String,                    token_mint: String,                       │
│    decimals: u8,                      reserve_a: Decimal,                       │
│    creator: String,                   reserve_b: Decimal,                       │
│    supply: Decimal,                   liquidity_sol: Decimal,                   │
│    created_slot: u64,                 spot_price: Decimal,                      │
│  }                                  }                                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse Ingestion for Enriched Trades

```sql
-- NATS Engine subscribes to enriched trades
CREATE TABLE datastreams.enriched_trades_queue (
    signature String,
    slot UInt64,
    transaction_index UInt64,
    instruction_index UInt8,
    trader String,
    market String,
    dex String,
    anchor_mint String,
    token_mint String,
    token_name String,
    token_symbol String,
    trade_direction UInt8,
    anchor_amount Decimal128(18),
    token_amount Decimal128(18),
    volume_sol Decimal128(18),
    volume_usd Decimal128(18),
    spot_price_sol Decimal128(18),
    spot_price_usd Decimal128(18),
    execution_price_sol Decimal128(18),
    execution_price_usd Decimal128(18),
    market_cap_sol Decimal128(18),
    market_cap_usd Decimal128(18),
    liquidity_sol Decimal128(18),
    liquidity_usd Decimal128(18),
    token_supply Decimal128(18),
    hops UInt8,
    hop_index UInt8,
    bonding_curve_pct Nullable(Decimal128(18)),
    transaction_fee_sol Decimal128(18),
    transaction_fee_usd Decimal128(18),
    priority_fee_sol Decimal128(18),
    priority_fee_usd Decimal128(18),
    validator_tip_sol Decimal128(18),
    validator_tip_usd Decimal128(18),
    dex_fee_sol Decimal128(18),
    dex_fee_usd Decimal128(18),
    platform_fee_sol Decimal128(18),
    platform_fee_usd Decimal128(18),
    block_timestamp DateTime64(9),
    server_timestamp DateTime64(9)
)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats:4222',
    nats_subjects = 'solana.enriched_trades',
    nats_format = 'RowBinary',
    nats_num_consumers = 4;

-- Materialized View routes to storage with fan-out
CREATE MATERIALIZED VIEW datastreams.enriched_trades_mv
TO datastreams.enriched_trades AS
SELECT *, now64(3) AS ingested_at
FROM datastreams.enriched_trades_queue;
```

---

## Data Pipeline: OHLCV Generation Flow

### How OHLCV Candles Are Generated

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    OHLCV GENERATION PIPELINE                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  INPUT: solana.enriched_trades (from enrich_trade transform)                    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                    WINDOWED AGGREGATION TRANSFORMS                           ││
│  │                                                                              ││
│  │  Each timeframe has a dedicated transform with tumbling windows:             ││
│  │                                                                              ││
│  │  ┌─────────────────────────────────────────────────────────────────────┐    ││
│  │  │ Transform: market_ohlcv_1s (Window: 1 second)                        │    ││
│  │  │ ├─ Partition By: market (process each pool independently)           │    ││
│  │  │ ├─ Window Start: Aligned to second boundary                         │    ││
│  │  │ ├─ On window close: Aggregate all trades in window                  │    ││
│  │  │ └─ Emit to: solana.market_ohlcv.1s                                  │    ││
│  │  └─────────────────────────────────────────────────────────────────────┘    ││
│  │                                                                              ││
│  │  ┌─────────────────────────────────────────────────────────────────────┐    ││
│  │  │ Transform: token_ohlcv_5m (Window: 5 minutes)                        │    ││
│  │  │ ├─ Partition By: token_mint (aggregate across all pools)            │    ││
│  │  │ ├─ Window Start: Aligned to 5-minute boundary                       │    ││
│  │  │ ├─ Aggregation: Merge trades from ALL pools for this token          │    ││
│  │  │ └─ Emit to: solana.token_ohlcv.5m                                   │    ││
│  │  └─────────────────────────────────────────────────────────────────────┘    ││
│  │                                                                              ││
│  │  Total: 30 transforms (15 timeframes × 2 levels)                            ││
│  │  ├─ Market OHLCV: Per-pool candles (for LP analytics)                       ││
│  │  └─ Token OHLCV: Cross-pool aggregate (for TradingView charts)              ││
│  │                                                                              ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│  AGGREGATION LOGIC (within each window):                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                                                                              ││
│  │  trades_in_window = [t1, t2, t3, ..., tN]                                   ││
│  │                                                                              ││
│  │  Ohlcv {                                                                     ││
│  │    o_price_sol: trades_in_window.first().spot_price_sol,                    ││
│  │    h_price_sol: trades_in_window.map(t => t.spot_price_sol).max(),         ││
│  │    l_price_sol: trades_in_window.map(t => t.spot_price_sol).min(),         ││
│  │    c_price_sol: trades_in_window.last().spot_price_sol,                     ││
│  │    v_sol: trades_in_window.map(t => t.volume_sol).sum(),                    ││
│  │    v_buy_sol: trades_in_window.filter(t => t.is_buy).sum(volume_sol),       ││
│  │    v_sell_sol: trades_in_window.filter(t => t.is_sell).sum(volume_sol),     ││
│  │    buys: trades_in_window.filter(t => t.is_buy).count(),                    ││
│  │    sells: trades_in_window.filter(t => t.is_sell).count(),                  ││
│  │    liquidity_sol: trades_in_window.last().liquidity_sol,                    ││
│  │    timestamp: window.start_time,                                             ││
│  │  }                                                                           ││
│  │                                                                              ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│  OUTPUT: 30 NATS subjects                                                       │
│  ├── solana.market_ohlcv.1s, .5s, .15s, .30s, .1m, ... .1w (15 subjects)       │
│  └── solana.token_ohlcv.1s, .5s, .15s, .30s, .1m, ... .1w (15 subjects)        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse Ingestion with Tier Routing

```sql
-- Single queue subscribes to ALL token OHLCV subjects using wildcard
CREATE TABLE datastreams.token_ohlcv_queue (
    token String,
    market String,  -- Empty for token-level OHLCV
    dex String,
    o_price_sol Decimal128(18),
    h_price_sol Decimal128(18),
    l_price_sol Decimal128(18),
    c_price_sol Decimal128(18),
    -- ... all 28 fields
    timestamp UInt64,
    _subject String  -- NATS provides this automatically
)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats:4222',
    nats_subjects = 'solana.token_ohlcv.*',  -- Wildcard subscription
    nats_format = 'RowBinary',
    nats_num_consumers = 4;

-- Route to SHORT tier (1s, 5s, 15s, 30s) - 7 day TTL
CREATE MATERIALIZED VIEW datastreams.token_ohlcv_short_mv
TO datastreams.token_ohlcv_short AS
SELECT
    token,
    extractAllGroups(_subject, 'solana\\.token_ohlcv\\.([^.]+)')[1][1] AS timeframe,
    fromUnixTimestamp64Milli(timestamp * 1000) AS candle_time,
    * EXCEPT (token, timestamp, _subject)
FROM datastreams.token_ohlcv_queue
WHERE extractAllGroups(_subject, 'solana\\.token_ohlcv\\.([^.]+)')[1][1]
    IN ('1s', '5s', '15s', '30s');

-- Route to MEDIUM tier (1m-1h) - 90 day TTL
CREATE MATERIALIZED VIEW datastreams.token_ohlcv_medium_mv
TO datastreams.token_ohlcv_medium AS
SELECT ...
WHERE ... IN ('1m', '3m', '5m', '15m', '30m', '1h');

-- Route to LONG tier (4h-1w) - Forever
CREATE MATERIALIZED VIEW datastreams.token_ohlcv_long_mv
TO datastreams.token_ohlcv_long AS
SELECT ...
WHERE ... IN ('4h', '6h', '12h', '1d', '1w');
```

---

## Theory: OHLCV Candlestick Aggregation

### What is OHLCV?

OHLCV is the standard for financial time-series visualization:

```
                    5-Minute Candle
         ┌─────────────────────────────────────┐
         │                                     │
  Price  │         ┬ High ($0.00085)          │
    ↑    │         │                          │
         │      ┌──┴──┐                        │
         │      │     │ ← Body (Open to Close) │
         │      │     │                        │
         │ Open │█████│ Close                  │
         │ $0.79│█████│ $0.82                  │
         │      │     │                        │
         │      └──┬──┘                        │
         │         │                          │
         │         ┴ Low ($0.00077)           │
         │                                     │
         │    Volume: 15,234 SOL              │
         │    Buys: 89 | Sells: 67            │
         └─────────────────────────────────────┘
              10:00                    10:05
```

### Aggregation Rules

| Metric | Aggregation | Formula |
|--------|-------------|---------|
| **Open** | FIRST | `argMin(price, timestamp)` |
| **High** | MAX | `max(price)` |
| **Low** | MIN | `min(price)` |
| **Close** | LAST | `argMax(price, timestamp)` |
| **Volume** | SUM | `sum(volume)` |
| **Buys** | COUNT | `countIf(direction = 'Buy')` |
| **Sells** | COUNT | `countIf(direction = 'Sell')` |
| **Liquidity** | LAST | `argMax(liquidity, timestamp)` |

### Market OHLCV vs Token OHLCV

| Type | Scope | Use Case |
|------|-------|----------|
| **Market OHLCV** | Single pool (SOL/BONK on Raydium) | LP analytics, pool-specific trading |
| **Token OHLCV** | Aggregate all pools for token | TradingView charts, token analytics |

---

## OHLCV Metrics (28 Fields per Candle)

### Complete OHLCV Field Reference

| # | Field | Type | Description |
|---|-------|------|-------------|
| 1 | `market` | String | Pool address (empty for token OHLCV) |
| 2 | `token` | String | Token mint address |
| 3 | `dex` | String | DEX name (empty for token OHLCV) |
| 4 | `o_price_sol` | Decimal128(18) | Open price in SOL |
| 5 | `h_price_sol` | Decimal128(18) | High price in SOL |
| 6 | `l_price_sol` | Decimal128(18) | Low price in SOL |
| 7 | `c_price_sol` | Decimal128(18) | Close price in SOL |
| 8 | `o_price_usd` | Decimal128(18) | Open price in USD |
| 9 | `h_price_usd` | Decimal128(18) | High price in USD |
| 10 | `l_price_usd` | Decimal128(18) | Low price in USD |
| 11 | `c_price_usd` | Decimal128(18) | Close price in USD |
| 12 | `o_mc_sol` | Decimal128(18) | Open market cap SOL |
| 13 | `h_mc_sol` | Decimal128(18) | High market cap SOL |
| 14 | `l_mc_sol` | Decimal128(18) | Low market cap SOL |
| 15 | `c_mc_sol` | Decimal128(18) | Close market cap SOL |
| 16 | `o_mc_usd` | Decimal128(18) | Open market cap USD |
| 17 | `h_mc_usd` | Decimal128(18) | High market cap USD |
| 18 | `l_mc_usd` | Decimal128(18) | Low market cap USD |
| 19 | `c_mc_usd` | Decimal128(18) | Close market cap USD |
| 20 | `v_sol` | Decimal128(18) | Total volume SOL |
| 21 | `v_usd` | Decimal128(18) | Total volume USD |
| 22 | `v_buy_sol` | Decimal128(18) | Buy volume SOL |
| 23 | `v_sell_sol` | Decimal128(18) | Sell volume SOL |
| 24 | `v_buy_usd` | Decimal128(18) | Buy volume USD |
| 25 | `v_sell_usd` | Decimal128(18) | Sell volume USD |
| 26 | `liquidity_sol` | Decimal128(18) | End-of-period liquidity SOL |
| 27 | `liquidity_usd` | Decimal128(18) | End-of-period liquidity USD |
| 28 | `buys` | UInt64 | Buy trade count |
| 29 | `sells` | UInt64 | Sell trade count |
| 30 | `timestamp` | UInt64 | Window start (Unix seconds) |

### 30 OHLCV Subjects (15 Timeframes × 2 Levels)

| Timeframe | Market Subject | Token Subject | Data Volume/Day | Retention |
|-----------|----------------|---------------|-----------------|-----------|
| 1s | `market_ohlcv.1s` | `token_ohlcv.1s` | ~864M rows | 7 days |
| 5s | `market_ohlcv.5s` | `token_ohlcv.5s` | ~173M rows | 7 days |
| 15s | `market_ohlcv.15s` | `token_ohlcv.15s` | ~58M rows | 7 days |
| 30s | `market_ohlcv.30s` | `token_ohlcv.30s` | ~29M rows | 7 days |
| 1m | `market_ohlcv.1m` | `token_ohlcv.1m` | ~14M rows | 90 days |
| 3m | `market_ohlcv.3m` | `token_ohlcv.3m` | ~4.8M rows | 90 days |
| 5m | `market_ohlcv.5m` | `token_ohlcv.5m` | ~2.9M rows | 90 days |
| 15m | `market_ohlcv.15m` | `token_ohlcv.15m` | ~960K rows | 90 days |
| 30m | `market_ohlcv.30m` | `token_ohlcv.30m` | ~480K rows | 90 days |
| 1h | `market_ohlcv.1h` | `token_ohlcv.1h` | ~240K rows | 90 days |
| 4h | `market_ohlcv.4h` | `token_ohlcv.4h` | ~60K rows | Forever |
| 6h | `market_ohlcv.6h` | `token_ohlcv.6h` | ~40K rows | Forever |
| 12h | `market_ohlcv.12h` | `token_ohlcv.12h` | ~20K rows | Forever |
| 1d | `market_ohlcv.1d` | `token_ohlcv.1d` | ~10K rows | Forever |
| 1w | `market_ohlcv.1w` | `token_ohlcv.1w` | ~1.4K rows | Forever |

---

## ClickHouse Deep Dive: MergeTree Engine

### Theory: How MergeTree Works

MergeTree is ClickHouse's core engine, designed for analytical workloads:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MergeTree Architecture                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  INSERT (batched, async)                                                │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│  │ Part 1  │  │ Part 2  │  │ Part 3  │  │ Part 4  │  ← Immutable parts │
│  │ (5MB)   │  │ (8MB)   │  │ (3MB)   │  │ (12MB)  │                    │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                    │
│       │            │            │            │                          │
│       └────────────┴─────┬──────┴────────────┘                          │
│                          │ Background merge                              │
│                          ▼                                               │
│                   ┌────────────┐                                         │
│                   │ Merged Part│  Sorted by ORDER BY                     │
│                   │ (28MB)     │  Duplicates preserved (or merged)       │
│                   └────────────┘                                         │
│                                                                          │
│  Part Contents:                                                          │
│  ├── primary.idx      Sparse index (every 8192 rows)                    │
│  ├── {column}.bin     Columnar data (compressed)                        │
│  ├── {column}.mrk2    Mark files (offsets into data)                    │
│  ├── count.txt        Row count                                          │
│  ├── columns.txt      Column list                                        │
│  └── skp_idx_*.idx    Data skipping indexes                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key MergeTree Concepts

| Concept | Description | Impact on Queries |
|---------|-------------|-------------------|
| **Parts** | Immutable data chunks | Lock-free reads, parallel scans |
| **Sparse Index** | Every Nth row (default 8192) | O(log N) to find data range |
| **Granules** | 8192 rows = 1 granule | Minimum read unit |
| **Background Merges** | Combines parts over time | Better compression, fewer files |
| **Columnar Storage** | Each column separate file | Only read needed columns |

### MergeTree Engine Variants

| Engine | Use Case | Behavior | Our Tables |
|--------|----------|----------|------------|
| **MergeTree** | Append-only logs | Keeps all rows | `enriched_trades`, `*_ohlcv_*` |
| **ReplacingMergeTree** | Latest state | Deduplicates by ORDER BY | `token_metrics`, `position_overview` |
| **AggregatingMergeTree** | Pre-computed aggregates | Merges aggregate states | `trending_*_daily` |
| **SummingMergeTree** | Running counters | Sums numeric columns | Volume counters |

---

## ClickHouse Optimization: ORDER BY Strategy

### Theory: ORDER BY = Physical Sort + Primary Key

In ClickHouse, `ORDER BY` determines:
1. **Physical sort order** of data on disk
2. **Primary key** for sparse index
3. **Query performance** for range scans

```sql
-- ORDER BY affects which queries are fast
CREATE TABLE trades (...) ORDER BY (token_mint, block_timestamp, signature)

-- FAST: Filters on ORDER BY prefix
SELECT * FROM trades
WHERE token_mint = 'ABC...'                    -- ✓ Uses index
  AND block_timestamp >= now() - INTERVAL 1 HOUR  -- ✓ Uses index

-- SLOW: Filter not in ORDER BY
SELECT * FROM trades
WHERE trader = 'XYZ...'                        -- ✗ Full scan!
```

### Choosing ORDER BY for Different Access Patterns

| Access Pattern | Optimal ORDER BY | Example Query |
|----------------|------------------|---------------|
| Token analytics | `(token_mint, block_timestamp)` | Get BONK trades last hour |
| Wallet history | `(trader, block_timestamp)` | Get all trades for wallet |
| Market depth | `(market, block_timestamp)` | Get Raydium SOL/BONK trades |
| Global timeline | `(block_timestamp)` | Get all trades last 5 min |

### Our Solution: Multiple Tables with Different ORDER BY

```sql
-- Primary table: Optimized for token queries (most common)
CREATE TABLE enriched_trades
ORDER BY (token_mint, block_timestamp, signature);

-- Secondary table: Optimized for wallet queries
CREATE MATERIALIZED VIEW enriched_trades_by_trader
ENGINE = MergeTree() ORDER BY (trader, block_timestamp, signature)
AS SELECT * FROM enriched_trades;

-- Secondary table: Optimized for market queries
CREATE MATERIALIZED VIEW enriched_trades_by_market
ENGINE = MergeTree() ORDER BY (market, block_timestamp, signature)
AS SELECT * FROM enriched_trades;
```

### ORDER BY Design Principles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ORDER BY Best Practices                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. MOST SELECTIVE FIRST                                                │
│     └─ (token_mint, ...) → 10,000+ unique values                        │
│     └─ NOT (trade_direction, ...) → only 2 values                       │
│                                                                          │
│  2. TIME AFTER ENTITY                                                   │
│     └─ (token_mint, block_timestamp) for time-range queries             │
│     └─ Enables efficient: WHERE token='X' AND time > Y                  │
│                                                                          │
│  3. UNIQUENESS SUFFIX                                                   │
│     └─ Add signature/id as final column                                 │
│     └─ Ensures deterministic ordering within same timestamp             │
│                                                                          │
│  4. AVOID HIGH-CARDINALITY AFTER LOW-CARDINALITY                       │
│     └─ BAD: (dex, trader, ...) → dex has 20 values                     │
│     └─ Trader column won't benefit from index pruning                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ClickHouse Optimization: Data Skipping Indexes

### Theory: Beyond the Primary Index

Primary index finds the right granules (8192-row chunks). Data skipping indexes let us skip entire granules based on per-granule statistics.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Data Skipping Index Mechanics                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Granule 0 (rows 0-8191):                                               │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ minmax(price_usd): [0.0001, 0.0015]                                │ │
│  │ set(dex): {'raydium', 'orca'}                                      │ │
│  │ bloom_filter(trader): {hash1, hash2, hash3, ...}                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Granule 1 (rows 8192-16383):                                           │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ minmax(price_usd): [0.0012, 0.0089]                                │ │
│  │ set(dex): {'pump', 'meteora'}                                      │ │
│  │ bloom_filter(trader): {hash4, hash5, hash6, ...}                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Query: WHERE dex = 'raydium' AND price_usd < 0.001                     │
│                                                                          │
│  Evaluation:                                                             │
│  - Granule 0: dex set contains 'raydium' ✓, price range overlaps ✓     │
│    → MUST READ                                                          │
│  - Granule 1: dex set doesn't contain 'raydium' ✗                       │
│    → SKIP (don't read at all)                                           │
│                                                                          │
│  Result: 50% of data skipped before reading any rows!                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Index Types and When to Use Them

| Index Type | Best For | Storage Overhead | False Positive Rate |
|------------|----------|------------------|---------------------|
| `minmax` | Range queries on ordered data | Tiny (2 values/granule) | None |
| `set(N)` | IN queries, low-cardinality | Small (N values/granule) | None |
| `bloom_filter(p)` | Equality on high-cardinality | Medium | ~p% |
| `tokenbf_v1` | LIKE queries on text | Medium | ~1-5% |
| `ngrambf_v1` | Substring search | Large | ~1-5% |

### Our Index Strategy for Trades

```sql
CREATE TABLE enriched_trades (
    -- Columns...

    -- ═══════════════════════════════════════════════════════════════════
    -- DATA SKIPPING INDEXES
    -- ═══════════════════════════════════════════════════════════════════

    -- For wallet queries (trader not in ORDER BY)
    -- bloom_filter(0.01) = 1% false positive rate
    INDEX idx_trader trader TYPE bloom_filter(0.01) GRANULARITY 4,

    -- For DEX filtering (only ~20 unique values)
    -- set(20) stores up to 20 unique values per granule
    INDEX idx_dex dex TYPE set(20) GRANULARITY 1,

    -- For price range queries
    INDEX idx_price_sol spot_price_sol TYPE minmax GRANULARITY 1,
    INDEX idx_price_usd spot_price_usd TYPE minmax GRANULARITY 1,

    -- For volume filtering
    INDEX idx_volume volume_usd TYPE minmax GRANULARITY 1,

    -- For market cap filtering
    INDEX idx_mcap market_cap_usd TYPE minmax GRANULARITY 1,

    -- For symbol search (LIKE '%PEPE%')
    INDEX idx_symbol token_symbol TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4,

    -- For bonding curve filtering
    INDEX idx_bonding bonding_curve_pct TYPE minmax GRANULARITY 1
);
```

### GRANULARITY Explained

```
GRANULARITY = How many granules share one index entry

GRANULARITY 1: Index entry per granule (finest)
├── Granule 0 → Index entry 0
├── Granule 1 → Index entry 1
└── Best for: minmax, set (small overhead)

GRANULARITY 4: Index entry per 4 granules (coarser)
├── Granules 0-3 → Index entry 0
├── Granules 4-7 → Index entry 1
└── Best for: bloom_filter, tokenbf (reduce overhead)
```

---

## ClickHouse Optimization: Tiered Storage & TTL

### Theory: Hot/Warm/Cold Data Architecture

Not all data has equal access frequency:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Data Temperature Tiers                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  HOT (< 7 days)           WARM (7-90 days)       COLD (> 90 days)       │
│  ┌────────────────┐       ┌────────────────┐     ┌────────────────┐     │
│  │ NVMe SSD       │       │ SATA SSD       │     │ S3 / HDD       │     │
│  │ ~1TB           │       │ ~5TB           │     │ ~Unlimited     │     │
│  │ 500K IOPS      │       │ 50K IOPS       │     │ ~100 IOPS      │     │
│  │ <1ms latency   │       │ ~5ms latency   │     │ ~100ms latency │     │
│  │ $$$            │       │ $$             │     │ $              │     │
│  └────────────────┘       └────────────────┘     └────────────────┘     │
│                                                                          │
│  Access Pattern:                                                         │
│  └─ 90% of queries hit last 7 days (hot)                                │
│  └─ 9% of queries hit 7-90 days (warm)                                  │
│  └─ 1% of queries hit >90 days (cold)                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Storage Policy Configuration

```xml
<!-- /etc/clickhouse-server/config.d/storage.xml -->
<clickhouse>
    <storage_configuration>
        <disks>
            <hot>
                <path>/data/clickhouse/hot/</path>
            </hot>
            <warm>
                <path>/data/clickhouse/warm/</path>
            </warm>
            <cold>
                <type>s3</type>
                <endpoint>https://s3.amazonaws.com/bucket/cold/</endpoint>
                <access_key_id>xxx</access_key_id>
                <secret_access_key>xxx</secret_access_key>
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
</clickhouse>
```

### TTL Expressions

```sql
CREATE TABLE enriched_trades (...)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature)
TTL
    -- Move to warm storage after 7 days
    block_timestamp + INTERVAL 7 DAY TO VOLUME 'warm',
    -- Move to cold storage after 90 days
    block_timestamp + INTERVAL 90 DAY TO VOLUME 'cold',
    -- Delete after 1 year
    block_timestamp + INTERVAL 365 DAY DELETE
SETTINGS
    storage_policy = 'tiered',
    ttl_only_drop_parts = 1;  -- Critical for performance!
```

### ttl_only_drop_parts = 1 Explained

```
Without ttl_only_drop_parts:
└─ ClickHouse rewrites parts, removing expired rows
└─ Expensive I/O operation
└─ Part: [expired_row, valid_row, expired_row] → [valid_row]

With ttl_only_drop_parts = 1:
└─ ClickHouse only drops entire parts where ALL rows expired
└─ No rewrite, just delete file
└─ Requires: Partition by time so parts align with TTL boundaries
```

### OHLCV Tiered Storage Strategy

```sql
-- TIER 1: Short-term (1s, 5s, 15s, 30s) - 7 day retention
CREATE TABLE token_ohlcv_short (...)
PARTITION BY toYYYYMMDD(candle_time)    -- Daily partitions
TTL candle_time + INTERVAL 7 DAY DELETE
SETTINGS ttl_only_drop_parts = 1;

-- TIER 2: Medium-term (1m, 5m, 15m, 30m, 1h) - 90 day retention
CREATE TABLE token_ohlcv_medium (...)
PARTITION BY toYYYYMM(candle_time)      -- Monthly partitions
TTL
    candle_time + INTERVAL 30 DAY TO VOLUME 'warm',
    candle_time + INTERVAL 90 DAY DELETE
SETTINGS ttl_only_drop_parts = 1;

-- TIER 3: Long-term (4h, 6h, 12h, 1d, 1w) - Forever
CREATE TABLE token_ohlcv_long (...)
PARTITION BY toYYYYMM(candle_time)
TTL candle_time + INTERVAL 365 DAY TO VOLUME 'cold';
-- No DELETE - keep forever
```

---

## ClickHouse Optimization: Compression Codecs

### Theory: Columnar Compression Advantages

ClickHouse stores data column-by-column, enabling column-specific compression:

```
Row-Oriented (MySQL):              Column-Oriented (ClickHouse):
┌─────────────────────────┐        ┌──────────────────────────────┐
│ Row 1: id=1, price=0.01 │        │ Column: id                   │
│ Row 2: id=2, price=0.02 │        │ [1, 2, 3, 4, 5...]          │
│ Row 3: id=3, price=0.01 │        │ Codec: Delta + LZ4           │
│ Row 4: id=4, price=0.03 │        │ Compression: 95%             │
│ ...                     │        ├──────────────────────────────┤
└─────────────────────────┘        │ Column: price                │
                                   │ [0.01, 0.02, 0.01, 0.03...] │
                                   │ Codec: Gorilla + LZ4         │
                                   │ Compression: 80%             │
                                   └──────────────────────────────┘
```

### Codec Selection Guide

| Data Pattern | Best Codec | Explanation |
|--------------|------------|-------------|
| Monotonic timestamps | `Delta + ZSTD` | Store differences, then compress |
| Sequential IDs | `Delta + LZ4` | Differences are small/constant |
| Floating-point prices | `Gorilla + LZ4` | XOR-based float compression |
| Small integers (counts) | `T64 + LZ4` | 64-bit aligned compression |
| Low-cardinality strings | `LowCardinality` | Dictionary encoding |
| Random strings (hashes) | `ZSTD(3)` | General-purpose compression |

### Our Compression Strategy

```sql
CREATE TABLE enriched_trades (
    -- Timestamps: Delta compression (differences are tiny)
    block_timestamp DateTime64(9) CODEC(Delta, ZSTD(3)),
    server_timestamp DateTime64(9) CODEC(Delta, ZSTD(3)),

    -- Slot: Delta (monotonically increasing)
    slot UInt64 CODEC(Delta, LZ4),

    -- Addresses/signatures: ZSTD (random strings)
    signature String CODEC(ZSTD(3)),
    trader String CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),

    -- Low-cardinality: Dictionary encoding (built-in)
    dex LowCardinality(String),          -- ~20 unique values
    trade_direction UInt8,               -- 0 or 1

    -- Prices: Gorilla (XOR-based float compression)
    spot_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    spot_price_usd Decimal128(18) CODEC(Gorilla, LZ4),

    -- Volumes: DoubleDelta (slowly changing values)
    volume_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    volume_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),

    -- Counts: T64 (small integers)
    buys UInt64 CODEC(T64, LZ4),
    sells UInt64 CODEC(T64, LZ4),
    hops UInt8 CODEC(T64, LZ4)
);
```

### Production Compression Results

| Column | Raw Size | Compressed | Ratio |
|--------|----------|------------|-------|
| `block_timestamp` | 8 bytes | 0.3 bytes | **27:1** |
| `signature` | 88 bytes | 38 bytes | 2.3:1 |
| `trader` | 44 bytes | 15 bytes | 2.9:1 |
| `dex` | 10 bytes | 0.15 bytes | **67:1** |
| `spot_price_sol` | 16 bytes | 2.8 bytes | 5.7:1 |
| `volume_sol` | 16 bytes | 1.9 bytes | 8.4:1 |
| **Total row average** | ~420 bytes | **~38 bytes** | **11:1** |

---

## Complete DDL: Enriched Trades

```sql
-- ════════════════════════════════════════════════════════════════════════════
-- ENRICHED TRADES TABLE
-- Purpose: Stores all enriched trade events with full metrics
-- Engine: ReplicatedMergeTree for high-availability
-- ════════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.enriched_trades ON CLUSTER '{cluster}'
(
    -- ════════════════════════════════════════════════════════════════════════
    -- TRANSACTION IDENTITY
    -- ════════════════════════════════════════════════════════════════════════
    signature String CODEC(ZSTD(3)) COMMENT 'Unique 88-char base58 tx signature',
    slot UInt64 CODEC(Delta, LZ4) COMMENT 'Solana slot number',
    transaction_index UInt64 CODEC(Delta, LZ4) COMMENT 'Position within block',
    instruction_index UInt8 CODEC(T64, LZ4) COMMENT 'Instruction within tx',
    trader String CODEC(ZSTD(3)) COMMENT 'Wallet public key',
    market String CODEC(ZSTD(3)) COMMENT 'Pool/AMM address',

    -- ════════════════════════════════════════════════════════════════════════
    -- TOKEN INFORMATION
    -- ════════════════════════════════════════════════════════════════════════
    dex LowCardinality(String) COMMENT 'DEX name: raydium, pump, orca, etc.',
    anchor_mint String CODEC(ZSTD(3)) COMMENT 'Anchor currency (SOL/USDC/USDT)',
    token_mint String CODEC(ZSTD(3)) COMMENT 'Token being traded',
    token_name String CODEC(ZSTD(3)) COMMENT 'Token name from metadata',
    token_symbol String CODEC(ZSTD(3)) COMMENT 'Token symbol',

    -- ════════════════════════════════════════════════════════════════════════
    -- TRADE DETAILS
    -- ════════════════════════════════════════════════════════════════════════
    trade_direction UInt8 COMMENT '0=Buy, 1=Sell',
    anchor_amount Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Amount of anchor',
    token_amount Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Amount of tokens',
    volume_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)) COMMENT 'Volume in SOL',
    volume_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)) COMMENT 'Volume in USD',

    -- ════════════════════════════════════════════════════════════════════════
    -- PRICING (Point-in-time)
    -- ════════════════════════════════════════════════════════════════════════
    spot_price_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Pre-trade spot price SOL',
    spot_price_usd Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Pre-trade spot price USD',
    execution_price_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Actual execution price SOL',
    execution_price_usd Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Actual execution price USD',

    -- ════════════════════════════════════════════════════════════════════════
    -- MARKET METRICS (Point-in-time)
    -- ════════════════════════════════════════════════════════════════════════
    market_cap_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Market cap at trade time SOL',
    market_cap_usd Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Market cap at trade time USD',
    liquidity_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Pool liquidity SOL',
    liquidity_usd Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Pool liquidity USD',
    token_supply Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Total token supply',

    -- ════════════════════════════════════════════════════════════════════════
    -- MULTI-HOP METADATA
    -- ════════════════════════════════════════════════════════════════════════
    hops UInt8 CODEC(T64, LZ4) COMMENT 'Total hops in route (1=direct)',
    hop_index UInt8 CODEC(T64, LZ4) COMMENT 'This hop position (1-indexed)',
    bonding_curve_pct Nullable(Decimal128(18)) COMMENT 'Pump.fun curve % (null if graduated)',

    -- ════════════════════════════════════════════════════════════════════════
    -- FEE TRACKING (10 fields)
    -- ════════════════════════════════════════════════════════════════════════
    transaction_fee_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Base tx fee',
    transaction_fee_usd Decimal128(18) CODEC(Gorilla, LZ4),
    priority_fee_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Priority fee (compute units)',
    priority_fee_usd Decimal128(18) CODEC(Gorilla, LZ4),
    validator_tip_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'MEV tip (Jito)',
    validator_tip_usd Decimal128(18) CODEC(Gorilla, LZ4),
    dex_fee_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'DEX/AMM fee',
    dex_fee_usd Decimal128(18) CODEC(Gorilla, LZ4),
    platform_fee_sol Decimal128(18) CODEC(Gorilla, LZ4) COMMENT 'Trading bot fee',
    platform_fee_usd Decimal128(18) CODEC(Gorilla, LZ4),

    -- ════════════════════════════════════════════════════════════════════════
    -- TIMESTAMPS
    -- ════════════════════════════════════════════════════════════════════════
    block_timestamp DateTime64(9) CODEC(Delta, ZSTD(3)) COMMENT 'Block consensus time',
    server_timestamp DateTime64(9) CODEC(Delta, ZSTD(3)) COMMENT 'Indexer receipt time',
    ingested_at DateTime64(3) DEFAULT now64(3) COMMENT 'ClickHouse ingestion time',

    -- ════════════════════════════════════════════════════════════════════════
    -- DATA SKIPPING INDEXES
    -- ════════════════════════════════════════════════════════════════════════
    INDEX idx_trader trader TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_dex dex TYPE set(20) GRANULARITY 1,
    INDEX idx_price_sol spot_price_sol TYPE minmax GRANULARITY 1,
    INDEX idx_price_usd spot_price_usd TYPE minmax GRANULARITY 1,
    INDEX idx_volume volume_usd TYPE minmax GRANULARITY 1,
    INDEX idx_mcap market_cap_usd TYPE minmax GRANULARITY 1,
    INDEX idx_symbol token_symbol TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4,
    INDEX idx_bonding bonding_curve_pct TYPE minmax GRANULARITY 1
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/enriched_trades', '{replica}')
PARTITION BY toYYYYMM(block_timestamp)
ORDER BY (token_mint, block_timestamp, signature)
TTL
    block_timestamp + INTERVAL 7 DAY TO VOLUME 'warm',
    block_timestamp + INTERVAL 90 DAY TO VOLUME 'cold',
    block_timestamp + INTERVAL 365 DAY DELETE
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered',
    ttl_only_drop_parts = 1
COMMENT 'Enriched trade events with full metrics. ~100K+ rows/second. 11:1 compression.';
```

---

## Complete DDL: OHLCV Tiered Tables

```sql
-- ════════════════════════════════════════════════════════════════════════════
-- TIER 1: SHORT-TERM OHLCV (1s, 5s, 15s, 30s)
-- 7-day retention, daily partitions for efficient TTL
-- ════════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.token_ohlcv_short ON CLUSTER '{cluster}'
(
    token String CODEC(ZSTD(3)),
    timeframe LowCardinality(String) COMMENT '1s, 5s, 15s, 30s',
    candle_time DateTime64(3) CODEC(Delta, ZSTD(3)),

    -- Price OHLC
    o_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_usd Decimal128(18) CODEC(Gorilla, LZ4),

    -- Market Cap OHLC
    o_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),

    -- Volume
    v_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),

    -- Liquidity & Counts
    liquidity_sol Decimal128(18) CODEC(Gorilla, LZ4),
    liquidity_usd Decimal128(18) CODEC(Gorilla, LZ4),
    buys UInt64 CODEC(T64, LZ4),
    sells UInt64 CODEC(T64, LZ4),

    -- Indexes
    INDEX idx_timeframe timeframe TYPE set(4) GRANULARITY 1
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/token_ohlcv_short', '{replica}')
PARTITION BY toYYYYMMDD(candle_time)  -- Daily partitions for 7-day TTL
ORDER BY (token, timeframe, candle_time)
TTL candle_time + INTERVAL 7 DAY DELETE
SETTINGS
    index_granularity = 8192,
    ttl_only_drop_parts = 1
COMMENT 'Short-term OHLCV (1s-30s). 7-day retention. ~1.1B candles/day.';


-- ════════════════════════════════════════════════════════════════════════════
-- TIER 2: MEDIUM-TERM OHLCV (1m, 3m, 5m, 15m, 30m, 1h)
-- 90-day retention, monthly partitions
-- ════════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.token_ohlcv_medium ON CLUSTER '{cluster}'
(
    -- Same schema as token_ohlcv_short
    token String CODEC(ZSTD(3)),
    timeframe LowCardinality(String) COMMENT '1m, 3m, 5m, 15m, 30m, 1h',
    candle_time DateTime64(3) CODEC(Delta, ZSTD(3)),

    o_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_usd Decimal128(18) CODEC(Gorilla, LZ4),

    o_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),

    v_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),

    liquidity_sol Decimal128(18) CODEC(Gorilla, LZ4),
    liquidity_usd Decimal128(18) CODEC(Gorilla, LZ4),
    buys UInt64 CODEC(T64, LZ4),
    sells UInt64 CODEC(T64, LZ4),

    INDEX idx_timeframe timeframe TYPE set(6) GRANULARITY 1
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/token_ohlcv_medium', '{replica}')
PARTITION BY toYYYYMM(candle_time)  -- Monthly partitions
ORDER BY (token, timeframe, candle_time)
TTL
    candle_time + INTERVAL 30 DAY TO VOLUME 'warm',
    candle_time + INTERVAL 90 DAY DELETE
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered',
    ttl_only_drop_parts = 1
COMMENT 'Medium-term OHLCV (1m-1h). 90-day retention. ~24M candles/day.';


-- ════════════════════════════════════════════════════════════════════════════
-- TIER 3: LONG-TERM OHLCV (4h, 6h, 12h, 1d, 1w)
-- Keep forever, tiered to cold storage after 1 year
-- ════════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.token_ohlcv_long ON CLUSTER '{cluster}'
(
    token String CODEC(ZSTD(3)),
    timeframe LowCardinality(String) COMMENT '4h, 6h, 12h, 1d, 1w',
    candle_time DateTime64(3) CODEC(Delta, ZSTD(3)),

    o_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_price_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_price_usd Decimal128(18) CODEC(Gorilla, LZ4),

    o_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_sol Decimal128(18) CODEC(Gorilla, LZ4),
    o_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    h_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    l_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),
    c_mc_usd Decimal128(18) CODEC(Gorilla, LZ4),

    v_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_sol Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_buy_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),
    v_sell_usd Decimal128(18) CODEC(DoubleDelta, ZSTD(1)),

    liquidity_sol Decimal128(18) CODEC(Gorilla, LZ4),
    liquidity_usd Decimal128(18) CODEC(Gorilla, LZ4),
    buys UInt64 CODEC(T64, LZ4),
    sells UInt64 CODEC(T64, LZ4),

    INDEX idx_timeframe timeframe TYPE set(5) GRANULARITY 1
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/token_ohlcv_long', '{replica}')
PARTITION BY toYYYYMM(candle_time)
ORDER BY (token, timeframe, candle_time)
TTL candle_time + INTERVAL 365 DAY TO VOLUME 'cold'
SETTINGS
    index_granularity = 8192,
    storage_policy = 'tiered'
COMMENT 'Long-term OHLCV (4h-1w). Forever retention. ~132K candles/day.';


-- ════════════════════════════════════════════════════════════════════════════
-- UNIFIED VIEW: Query all tiers transparently
-- ════════════════════════════════════════════════════════════════════════════

CREATE VIEW datastreams.token_ohlcv AS
SELECT * FROM datastreams.token_ohlcv_short
UNION ALL
SELECT * FROM datastreams.token_ohlcv_medium
UNION ALL
SELECT * FROM datastreams.token_ohlcv_long;
```

---

## Materialized View Patterns

### NATS Queue → Storage Table Pattern

```sql
-- Step 1: NATS Engine queue (virtual table)
CREATE TABLE datastreams.enriched_trades_queue (
    -- Same schema as enriched_trades, but no indexes/TTL
    signature String,
    trader String,
    -- ... all fields
)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats:4222',
    nats_subjects = 'solana.enriched_trades',
    nats_format = 'RowBinary',
    nats_num_consumers = 4;  -- Parallel consumers

-- Step 2: Materialized View routes to storage
CREATE MATERIALIZED VIEW datastreams.enriched_trades_mv
TO datastreams.enriched_trades AS
SELECT
    *,
    now64(3) AS ingested_at
FROM datastreams.enriched_trades_queue;
```

### Fan-Out Pattern (One Queue → Multiple Tables)

```sql
-- One queue feeds multiple destinations
CREATE MATERIALIZED VIEW enriched_trades_main_mv TO enriched_trades
AS SELECT * FROM enriched_trades_queue;

CREATE MATERIALIZED VIEW enriched_trades_by_trader_mv TO enriched_trades_by_trader
AS SELECT * FROM enriched_trades_queue;

CREATE MATERIALIZED VIEW enriched_trades_by_market_mv TO enriched_trades_by_market
AS SELECT * FROM enriched_trades_queue;

CREATE MATERIALIZED VIEW enriched_trades_hourly_agg_mv TO enriched_trades_hourly
AS SELECT
    toStartOfHour(block_timestamp) AS hour,
    token_mint,
    dex,
    sum(volume_sol) AS total_volume_sol,
    count() AS trade_count
FROM enriched_trades_queue
GROUP BY hour, token_mint, dex;
```

### OHLCV Routing by Timeframe

```sql
-- Route to appropriate tier based on timeframe in subject
CREATE MATERIALIZED VIEW token_ohlcv_short_mv TO token_ohlcv_short AS
SELECT
    token,
    extractAllGroups(_subject, 'solana\\.token_ohlcv\\.([^.]+)')[1][1] AS timeframe,
    fromUnixTimestamp64Milli(timestamp * 1000) AS candle_time,
    *
FROM token_ohlcv_queue
WHERE extractAllGroups(_subject, 'solana\\.token_ohlcv\\.([^.]+)')[1][1]
    IN ('1s', '5s', '15s', '30s');

CREATE MATERIALIZED VIEW token_ohlcv_medium_mv TO token_ohlcv_medium AS
SELECT ...
WHERE ... IN ('1m', '3m', '5m', '15m', '30m', '1h');

CREATE MATERIALIZED VIEW token_ohlcv_long_mv TO token_ohlcv_long AS
SELECT ...
WHERE ... IN ('4h', '6h', '12h', '1d', '1w');
```

---

## Query Patterns & Performance

### TradingView Candle Query

```sql
-- Get last 500 5-minute candles for TradingView
-- Expected: < 30ms
SELECT
    toUnixTimestamp(candle_time) AS time,
    toFloat64(o_price_usd) AS open,
    toFloat64(h_price_usd) AS high,
    toFloat64(l_price_usd) AS low,
    toFloat64(c_price_usd) AS close,
    toFloat64(v_usd) AS volume
FROM datastreams.token_ohlcv_medium
WHERE token = 'DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263'
  AND timeframe = '5m'
ORDER BY candle_time DESC
LIMIT 500;
```

### Volume Analytics by DEX

```sql
-- DEX breakdown last 24h
-- Expected: < 100ms
SELECT
    dex,
    count() AS trades,
    sum(volume_usd) AS volume_usd,
    round(avg(volume_usd), 2) AS avg_trade_size,
    countIf(trade_direction = 0) AS buys,
    countIf(trade_direction = 1) AS sells,
    round(sum(dex_fee_sol), 4) AS total_dex_fees
FROM datastreams.enriched_trades
WHERE block_timestamp >= now() - INTERVAL 24 HOUR
GROUP BY dex
ORDER BY volume_usd DESC;
```

### Fee Analysis

```sql
-- Hourly fee breakdown by category
SELECT
    toStartOfHour(block_timestamp) AS hour,
    round(sum(transaction_fee_sol), 4) AS tx_fees,
    round(sum(priority_fee_sol), 4) AS priority_fees,
    round(sum(validator_tip_sol), 4) AS mev_tips,
    round(sum(dex_fee_sol), 4) AS dex_fees,
    round(sum(platform_fee_sol), 4) AS platform_fees
FROM datastreams.enriched_trades
WHERE block_timestamp >= now() - INTERVAL 7 DAY
  AND hop_index = 1  -- Only count per-tx fees once
GROUP BY hour
ORDER BY hour;
```

### Slippage Analysis

```sql
-- Average slippage by market cap tier
SELECT
    multiIf(
        market_cap_usd < 100000, 'micro (<$100K)',
        market_cap_usd < 1000000, 'small ($100K-$1M)',
        market_cap_usd < 10000000, 'medium ($1M-$10M)',
        'large (>$10M)'
    ) AS mcap_tier,
    round(avg(abs(execution_price_sol - spot_price_sol) / spot_price_sol * 100), 4) AS avg_slippage_pct,
    round(quantile(0.95)(abs(execution_price_sol - spot_price_sol) / spot_price_sol * 100), 4) AS p95_slippage_pct,
    count() AS trades
FROM datastreams.enriched_trades
WHERE block_timestamp >= now() - INTERVAL 1 HOUR
  AND spot_price_sol > 0
GROUP BY mcap_tier
ORDER BY avg_slippage_pct DESC;
```

---

## Summary: Metrics Covered in Part 2

### Trade Metrics (45 fields)
- Transaction identity (6)
- Token information (5)
- Trade details (5)
- Pricing (4)
- Market metrics (5)
- Multi-hop metadata (3)
- Fee tracking (10)
- Timestamps (3)
- Platform metadata (4)

### OHLCV Metrics (30 fields × 30 subjects = 900 unique data points)
- Price OHLC × 2 currencies (8)
- Market Cap OHLC × 2 currencies (8)
- Volume breakdown (6)
- Liquidity (2)
- Trade counts (2)
- Metadata (4)

### ClickHouse Optimizations Covered
- MergeTree engine selection
- ORDER BY strategy for different access patterns
- Data skipping indexes (bloom_filter, set, minmax, tokenbf)
- Tiered storage (hot/warm/cold)
- TTL policies with `ttl_only_drop_parts`
- Compression codecs (Delta, Gorilla, T64, ZSTD, LZ4)
- Materialized view patterns

---

## What's Next

**Part 3: Token Metrics, Rolling Windows & AggregatingMergeTree** covers:
- Rolling window statistics (5m, 1h, 6h, 24h)
- The 89-field TokenMetric struct
- Holder distribution metrics
- Risk metrics (snipers, bundlers, insiders)
- Pro trader detection
- AggregatingMergeTree for pre-computed aggregates

---

## Implementation References

| Document | Content |
|----------|---------|
| [08_ENRICHED_TRADES_SETUP.md](./08_ENRICHED_TRADES_SETUP.md) | Complete Trade DDL |
| [10_OHLCV_SETUP.md](./10_OHLCV_SETUP.md) | OHLCV tiered architecture |
| [04_CLICKHOUSE_OPTIMIZATION.md](./04_CLICKHOUSE_OPTIMIZATION.md) | Performance tuning guide |
| [17_DATA_FLOW.md](./17_DATA_FLOW.md) | Pipeline architecture |

---

*Part 2 of 6 in the "Building Real-Time Solana DEX Analytics with ClickHouse" series.*
