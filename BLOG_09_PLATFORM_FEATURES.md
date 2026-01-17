# Part 9: Platform Features - Orders, Rewards & Referrals

*Building user engagement systems with order tracking, tiered rewards, and multi-level referral earnings*

---

## Introduction

Beyond core trading analytics, a successful trading platform needs robust user engagement systems. In this final part of the series, we explore Click platform-specific features that drive user retention and growth:

1. **Order Management** - Market, limit, DCA, stop-loss, and take-profit order tracking
2. **User Rewards** - Points accumulation, tier progression, and cashback systems
3. **Referral Earnings** - Multi-tier (L1-L4) referral network with earnings attribution

These features share common ClickHouse patterns: ReplacingMergeTree for current state, MergeTree for history, and materialized views for real-time updates.

---

## Platform Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       CLICK PLATFORM ANALYTICS                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                       USER INTERACTIONS                                   │  │
│   │                                                                           │  │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                  │  │
│   │   │   TRADES    │    │   ORDERS    │    │  REFERRALS  │                  │  │
│   │   │             │    │             │    │             │                  │  │
│   │   │  Swaps,     │    │  Market,    │    │  L1-L4      │                  │  │
│   │   │  Fills      │    │  Limit,     │    │  Network    │                  │  │
│   │   │             │    │  DCA        │    │             │                  │  │
│   │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                  │  │
│   │          │                  │                  │                          │  │
│   └──────────┼──────────────────┼──────────────────┼──────────────────────────┘  │
│              │                  │                  │                             │
│              ▼                  ▼                  ▼                             │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                        DATASTREAMS                                        │  │
│   │                                                                           │  │
│   │   Aggregates user activity into structured events:                       │  │
│   │                                                                           │  │
│   │   • OrderOverview ───────► click.order_overview                          │  │
│   │   • UserRewardsOverview ─► click.user_rewards                            │  │
│   │   • ReferralEarningsOverview ► click.referral_earnings                   │  │
│   │   • ReferralEarning ─────► referral.earnings (individual events)         │  │
│   │                                                                           │  │
│   └──────────────────────────────────┬───────────────────────────────────────┘  │
│                                      │                                          │
│                                      ▼                                          │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                        CLICKHOUSE STORAGE                                 │  │
│   │                                                                           │  │
│   │   ┌─────────────────────────────────────────────────────────────────┐   │  │
│   │   │                     CURRENT STATE                                │   │  │
│   │   │                  (ReplacingMergeTree)                            │   │  │
│   │   │                                                                   │   │  │
│   │   │   order_overview    user_rewards    referral_earnings           │   │  │
│   │   │   (by order_id)     (by user_id)    (by referrer_user_id)       │   │  │
│   │   └─────────────────────────────────────────────────────────────────┘   │  │
│   │                                                                           │  │
│   │   ┌─────────────────────────────────────────────────────────────────┐   │  │
│   │   │                     HISTORY / EVENTS                             │   │  │
│   │   │                      (MergeTree)                                 │   │  │
│   │   │                                                                   │   │  │
│   │   │   order_history  user_rewards_history  referral_earning_events  │   │  │
│   │   │   (audit trail)  (tier progression)    (individual earnings)    │   │  │
│   │   └─────────────────────────────────────────────────────────────────┘   │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Order Management System

### Order Types and States

Click supports sophisticated order types beyond simple market orders:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          ORDER TYPE TAXONOMY                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                     IMMEDIATE EXECUTION                                  │   │
│   │                                                                           │   │
│   │   Market Order (0)                                                       │   │
│   │   • Executes immediately at current market price                        │   │
│   │   • May have slippage                                                    │   │
│   │   • Single or multiple fills possible                                    │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                     CONDITIONAL EXECUTION                                │   │
│   │                                                                           │   │
│   │   Limit Buy (1)          Limit Sell (2)                                 │   │
│   │   • Executes when price  • Executes when price                         │   │
│   │     drops to target        rises to target                              │   │
│   │   • Based on market cap  • Based on market cap                          │   │
│   │                                                                           │   │
│   │   Stop Loss (3)          Take Profit (4)                                │   │
│   │   • Sells when price     • Sells when price                             │   │
│   │     drops below threshold  rises above threshold                        │   │
│   │   • Risk management      • Lock in gains                                 │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                     RECURRING EXECUTION                                  │   │
│   │                                                                           │   │
│   │   DCA - Dollar Cost Averaging (5)                                       │   │
│   │   • Multiple trades over time                                            │   │
│   │   • Fixed amount per interval                                            │   │
│   │   • Smooths entry price                                                  │   │
│   │   • Many fills per order                                                 │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Order Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          ORDER STATE MACHINE                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                    ┌───────────────┐                                            │
│                    │    PENDING    │ ─────► Order created, waiting              │
│                    │     (0)       │        for conditions                       │
│                    └───────┬───────┘                                            │
│                            │                                                     │
│         ┌──────────────────┼──────────────────┐                                 │
│         │                  │                  │                                  │
│         ▼                  ▼                  ▼                                  │
│   ┌───────────┐    ┌─────────────┐    ┌────────────┐                           │
│   │ PARTIALLY │    │   FILLED    │    │ CANCELLED  │                           │
│   │  FILLED   │    │    (2)      │    │    (3)     │                           │
│   │   (1)     │    │             │    │            │                           │
│   └─────┬─────┘    └─────────────┘    └────────────┘                           │
│         │                 ▲                  ▲                                   │
│         │                 │                  │                                   │
│         └─────────────────┘                  │                                   │
│             (more fills)                     │                                   │
│                                              │                                   │
│   ┌─────────────────────────────────────────┘                                   │
│   │                                                                              │
│   ▼                                                                              │
│   ┌───────────┐    ┌───────────┐                                                │
│   │  EXPIRED  │    │  FAILED   │                                                │
│   │   (4)     │    │   (5)     │                                                │
│   └───────────┘    └───────────┘                                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Struct Definition

```rust
/// A Click user's order containing aggregated trade information for a specific token.
/// An order can consist of zero (if not yet filled) to multiple trades
/// (e.g., DCA, partial fills) in one trade direction. (only buys or only sells)
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct OrderOverview {
    /// Unique order identifier
    pub order_id: Uuid,

    /// Associated Click user ID
    pub user_id: String,

    /// Order type: Market(0), LimitBuy(1), LimitSell(2), StopLoss(3), TakeProfit(4), DCA(5)
    pub order_type: OrderType,

    /// Current status: Pending(0), PartiallyFilled(1), Filled(2), Cancelled(3), Expired(4), Failed(5)
    pub order_status: OrderStatus,

    /// Trade direction: Buy(0), Sell(1)
    pub trade_direction: TradeDirection,

    /// The input amount for the order (SOL for buy, tokens for sell)
    pub amount_in: Decimal,

    /// Token identification
    pub token_mint: String,
    pub token_name: String,
    pub token_symbol: String,

    /// Volume swapped (cumulative across all fills)
    pub volume_tokens: Decimal,
    pub volume_sol: Decimal,
    pub volume_usd: Decimal,

    /// Trigger condition for limit orders (market cap threshold)
    pub trigger_market_cap_usd: Option<Decimal>,

    /// Weighted average execution metrics
    pub avg_execution_price_sol: Decimal,
    pub avg_execution_price_usd: Decimal,
    pub avg_execution_market_cap_sol: Decimal,
    pub avg_execution_market_cap_usd: Decimal,

    /// Number of trades executed for this order
    pub trade_count: u64,

    /// Last update timestamp
    pub timestamp: OffsetDateTime,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- ORDER OVERVIEW - CURRENT STATE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Track latest state of each order
-- Engine: ReplacingMergeTree - One row per order, latest version wins
-- Updates: On every fill, status change, or cancellation
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.order_overview ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- ORDER IDENTIFIERS
    -- ═══════════════════════════════════════════════════════════════════════

    order_id UUID COMMENT 'Unique order identifier',
    user_id String COMMENT 'Click user ID',
    timestamp DateTime64(9) COMMENT 'Last update timestamp (version)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- ORDER CONFIGURATION
    -- ═══════════════════════════════════════════════════════════════════════

    order_type UInt8 COMMENT '0=Market, 1=LimitBuy, 2=LimitSell, 3=StopLoss, 4=TakeProfit, 5=DCA',
    order_status UInt8 COMMENT '0=Pending, 1=PartiallyFilled, 2=Filled, 3=Cancelled, 4=Expired, 5=Failed',
    trade_direction UInt8 COMMENT '0=Buy, 1=Sell',
    amount_in Decimal128(18) COMMENT 'Input amount (SOL for buy, tokens for sell)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TOKEN INFORMATION
    -- ═══════════════════════════════════════════════════════════════════════

    token_mint String COMMENT 'Token mint address',
    token_name String COMMENT 'Token name',
    token_symbol LowCardinality(String) COMMENT 'Token symbol',

    -- ═══════════════════════════════════════════════════════════════════════
    -- FILL METRICS (Cumulative)
    -- ═══════════════════════════════════════════════════════════════════════

    volume_tokens Decimal128(18) COMMENT 'Total tokens filled',
    volume_sol Decimal128(18) COMMENT 'Total SOL filled',
    volume_usd Decimal128(18) COMMENT 'Total USD filled',
    trade_count UInt64 COMMENT 'Number of fills',

    -- ═══════════════════════════════════════════════════════════════════════
    -- LIMIT ORDER CONFIGURATION
    -- ═══════════════════════════════════════════════════════════════════════

    trigger_market_cap_usd Nullable(Decimal128(18)) COMMENT 'Trigger market cap for limit orders',

    -- ═══════════════════════════════════════════════════════════════════════
    -- EXECUTION METRICS (Weighted Averages)
    -- ═══════════════════════════════════════════════════════════════════════

    avg_execution_price_sol Decimal128(18) COMMENT 'VWAP in SOL',
    avg_execution_price_usd Decimal128(18) COMMENT 'VWAP in USD',
    avg_execution_market_cap_sol Decimal128(18) COMMENT 'Avg market cap at execution (SOL)',
    avg_execution_market_cap_usd Decimal128(18) COMMENT 'Avg market cap at execution (USD)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Order type as string
    order_type_str LowCardinality(String) MATERIALIZED
        CASE order_type
            WHEN 0 THEN 'Market'
            WHEN 1 THEN 'LimitBuy'
            WHEN 2 THEN 'LimitSell'
            WHEN 3 THEN 'StopLoss'
            WHEN 4 THEN 'TakeProfit'
            WHEN 5 THEN 'DCA'
            ELSE 'Unknown'
        END,

    -- Order status as string
    order_status_str LowCardinality(String) MATERIALIZED
        CASE order_status
            WHEN 0 THEN 'Pending'
            WHEN 1 THEN 'PartiallyFilled'
            WHEN 2 THEN 'Filled'
            WHEN 3 THEN 'Cancelled'
            WHEN 4 THEN 'Expired'
            WHEN 5 THEN 'Failed'
            ELSE 'Unknown'
        END,

    -- Direction as string
    direction_str LowCardinality(String) MATERIALIZED
        if(trade_direction = 0, 'Buy', 'Sell'),

    -- Is order in terminal state?
    is_terminal Bool MATERIALIZED order_status IN (2, 3, 4, 5),

    -- Fill percentage
    fill_pct Decimal128(18) MATERIALIZED
        if(amount_in > 0,
           if(trade_direction = 0, (volume_sol / amount_in) * 100, (volume_tokens / amount_in) * 100),
           0),

    -- Ingestion timestamp
    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/order_overview',
    '{replica}',
    timestamp
)
PARTITION BY toYYYYMM(timestamp)
ORDER BY (order_id)
COMMENT 'Order current state. One row per order with latest status.'
SETTINGS index_granularity = 8192;
```

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- ORDER HISTORY - AUDIT TRAIL
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Track all order state changes for audit trail
-- Engine: MergeTree - Keeps all historical records
-- Use Case: "Show me how this DCA order filled over time"
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.order_history ON CLUSTER `server-apex`
(
    order_id UUID,
    user_id String,
    order_type UInt8,
    order_status UInt8,
    trade_direction UInt8,
    amount_in Decimal128(18),
    token_mint String,
    token_symbol LowCardinality(String),
    volume_tokens Decimal128(18),
    volume_sol Decimal128(18),
    volume_usd Decimal128(18),
    trade_count UInt64,
    avg_execution_price_usd Decimal128(18),
    timestamp DateTime64(9),
    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/order_history', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (user_id, order_id, timestamp)
TTL timestamp + INTERVAL 1 YEAR
COMMENT 'Order history for audit trail and DCA progress tracking. TTL: 1 year.'
SETTINGS index_granularity = 8192;
```

### Order Analytics Queries

**User's active orders:**
```sql
SELECT
    order_id,
    token_symbol,
    order_type_str,
    order_status_str,
    direction_str,
    amount_in,
    volume_sol,
    fill_pct,
    trade_count,
    trigger_market_cap_usd,
    timestamp
FROM datastreams.order_overview FINAL
WHERE user_id = {user_id:String}
  AND NOT is_terminal  -- Only active orders
ORDER BY timestamp DESC;
```

**DCA order progress over time:**
```sql
SELECT
    toStartOfHour(timestamp) AS hour,
    trade_count,
    volume_sol,
    volume_usd,
    avg_execution_price_usd,
    order_status
FROM datastreams.order_history
WHERE order_id = {order_id:UUID}
ORDER BY timestamp ASC;
```

**Order type distribution (last 24h):**
```sql
SELECT
    order_type_str,
    direction_str,
    count() AS order_count,
    sum(volume_usd) AS total_volume_usd,
    avg(trade_count) AS avg_fills_per_order,
    countIf(order_status = 2) AS filled_count,
    countIf(order_status = 3) AS cancelled_count,
    avg(fill_pct) AS avg_fill_pct
FROM datastreams.order_overview FINAL
WHERE timestamp >= now() - INTERVAL 24 HOUR
GROUP BY order_type_str, direction_str
ORDER BY total_volume_usd DESC;
```

**Limit orders waiting to trigger:**
```sql
SELECT
    order_id,
    user_id,
    token_symbol,
    direction_str,
    trigger_market_cap_usd,
    amount_in,
    dateDiff('hour', timestamp, now()) AS hours_waiting
FROM datastreams.order_overview FINAL
WHERE order_type IN (1, 2, 3, 4)  -- Limit orders
  AND order_status = 0  -- Pending
ORDER BY hours_waiting DESC
LIMIT 50;
```

---

## 2. User Rewards System

### Reward Mechanics

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         USER REWARDS SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                      POINTS ACCUMULATION                                 │   │
│   │                                                                           │   │
│   │   Points from Own Trades:                                                │   │
│   │   • Earned on every trade executed                                       │   │
│   │   • Rate varies by tier level                                            │   │
│   │   • Higher volume = more points                                          │   │
│   │                                                                           │   │
│   │   Points from Referrals:                                                 │   │
│   │   • Earned when referral network trades                                  │   │
│   │   • Multi-tier attribution (L1-L4)                                       │   │
│   │   • Bonus multipliers for active referrers                               │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│                                      ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                       TIER PROGRESSION                                   │   │
│   │                                                                           │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │   │
│   │   │ Bronze  │─▶│ Silver  │─▶│  Gold   │─▶│Platinum │─▶│ Diamond │      │   │
│   │   │ (Lvl 1) │  │ (Lvl 2) │  │ (Lvl 3) │  │ (Lvl 4) │  │ (Lvl 5) │      │   │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │   │
│   │                                                                           │   │
│   │   Benefits increase with tier:                                           │   │
│   │   • Higher cashback rates                                                │   │
│   │   • Better referral earnings                                             │   │
│   │   • Exclusive features                                                   │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│                                      ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                        CASHBACK REWARDS                                  │   │
│   │                                                                           │   │
│   │   Claimable Cashback:                                                    │   │
│   │   • Accumulated from trading fees                                        │   │
│   │   • Rate based on current tier                                           │   │
│   │   • Claimable anytime in SOL                                             │   │
│   │                                                                           │   │
│   │   Claimed All-Time:                                                      │   │
│   │   • Historical record of all claims                                      │   │
│   │   • Used for total earned calculation                                    │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Struct Definition

```rust
/// User rewards overview - aggregates points and cashback earned by a user
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct UserRewardsOverview {
    /// Unique Click user identifier
    pub user_id: String,

    /// Primary wallet address for the user
    pub address: String,

    /// Points earned from user's own trading activity
    pub points_from_own_trades: Decimal,

    /// Points earned from referral network activity
    pub points_from_referrals: Decimal,

    /// Total points (own + referral)
    pub total_points: Decimal,

    /// Cashback available to claim (in SOL)
    pub claimable_cashback: Decimal,

    /// Total cashback claimed historically (in SOL)
    pub claimed_all_time: Decimal,

    /// Current tier name (e.g., "Bronze", "Silver", "Gold")
    pub current_tier: String,

    /// Numeric tier level (1-5)
    pub tier_level: u8,

    /// Progress percentage toward next tier (0-100)
    pub tier_progress_pct: Decimal,

    /// Total number of trades executed
    pub total_trades: u64,

    /// Total trading volume
    pub total_volume_sol: Decimal,
    pub total_volume_usd: Decimal,

    /// Timestamp of last update
    pub last_updated: OffsetDateTime,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- USER REWARDS - CURRENT STATE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Track current rewards status for each user
-- Engine: ReplacingMergeTree - One row per user
-- Updates: On every trade, point earning, or cashback event
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.user_rewards ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- USER IDENTIFICATION
    -- ═══════════════════════════════════════════════════════════════════════

    user_id String COMMENT 'Unique Click user identifier',
    address String COMMENT 'Primary wallet address',
    last_updated DateTime64(9) COMMENT 'Last update timestamp (version)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- POINTS BREAKDOWN
    -- ═══════════════════════════════════════════════════════════════════════

    points_from_own_trades Decimal128(18) COMMENT 'Points from personal trading',
    points_from_referrals Decimal128(18) COMMENT 'Points from referral network',
    total_points Decimal128(18) COMMENT 'Total accumulated points',

    -- ═══════════════════════════════════════════════════════════════════════
    -- CASHBACK STATUS
    -- ═══════════════════════════════════════════════════════════════════════

    claimable_cashback Decimal128(18) COMMENT 'Available to claim (SOL)',
    claimed_all_time Decimal128(18) COMMENT 'Historically claimed (SOL)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TIER STATUS
    -- ═══════════════════════════════════════════════════════════════════════

    current_tier LowCardinality(String) COMMENT 'Tier name (Bronze, Silver, Gold, Platinum, Diamond)',
    tier_level UInt8 COMMENT 'Numeric tier level (1-5)',
    tier_progress_pct Decimal128(18) COMMENT 'Progress to next tier (0-100)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TRADING METRICS
    -- ═══════════════════════════════════════════════════════════════════════

    total_trades UInt64 COMMENT 'Total trade count',
    total_volume_sol Decimal128(18) COMMENT 'Total volume in SOL',
    total_volume_usd Decimal128(18) COMMENT 'Total volume in USD',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Total cashback ever earned
    total_earned_cashback Decimal128(18) MATERIALIZED claimable_cashback + claimed_all_time,

    -- Referral contribution percentage
    referral_points_pct Decimal128(18) MATERIALIZED
        if(total_points > 0, (points_from_referrals / total_points) * 100, 0),

    -- Average trade size
    avg_trade_size_sol Decimal128(18) MATERIALIZED
        if(total_trades > 0, total_volume_sol / total_trades, 0),

    -- Cashback rate (effective % of volume returned)
    cashback_rate_pct Decimal128(18) MATERIALIZED
        if(total_volume_sol > 0, ((claimable_cashback + claimed_all_time) / total_volume_sol) * 100, 0),

    -- Volume category
    volume_category LowCardinality(String) MATERIALIZED
        CASE
            WHEN total_volume_usd < 1000 THEN 'Starter'
            WHEN total_volume_usd < 10000 THEN 'Active'
            WHEN total_volume_usd < 100000 THEN 'Power'
            WHEN total_volume_usd < 1000000 THEN 'Whale'
            ELSE 'Mega Whale'
        END,

    -- Next tier name
    next_tier LowCardinality(String) MATERIALIZED
        CASE tier_level
            WHEN 1 THEN 'Silver'
            WHEN 2 THEN 'Gold'
            WHEN 3 THEN 'Platinum'
            WHEN 4 THEN 'Diamond'
            ELSE 'Max Tier'
        END,

    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/user_rewards',
    '{replica}',
    last_updated
)
ORDER BY (user_id)
COMMENT 'User rewards current state. One row per user.'
SETTINGS index_granularity = 8192;
```

### User Rewards Queries

**User's current rewards status:**
```sql
SELECT
    user_id,
    current_tier,
    tier_level,
    tier_progress_pct,
    next_tier,
    total_points,
    points_from_own_trades,
    points_from_referrals,
    referral_points_pct,
    claimable_cashback,
    claimed_all_time,
    total_earned_cashback,
    total_volume_usd,
    volume_category
FROM datastreams.user_rewards FINAL
WHERE user_id = {user_id:String};
```

**Points leaderboard:**
```sql
SELECT
    user_id,
    current_tier,
    tier_level,
    total_points,
    points_from_own_trades,
    points_from_referrals,
    referral_points_pct,
    total_volume_usd,
    total_trades
FROM datastreams.user_rewards FINAL
ORDER BY total_points DESC
LIMIT 100;
```

**Users with claimable cashback:**
```sql
SELECT
    user_id,
    address,
    claimable_cashback,
    claimed_all_time,
    total_earned_cashback,
    cashback_rate_pct,
    current_tier
FROM datastreams.user_rewards FINAL
WHERE claimable_cashback > 0.01  -- Minimum threshold
ORDER BY claimable_cashback DESC
LIMIT 50;
```

**Tier distribution:**
```sql
SELECT
    current_tier,
    tier_level,
    count() AS user_count,
    sum(total_points) AS total_points,
    sum(total_volume_usd) AS total_volume_usd,
    sum(claimable_cashback) AS pending_cashback,
    avg(tier_progress_pct) AS avg_progress_to_next
FROM datastreams.user_rewards FINAL
GROUP BY current_tier, tier_level
ORDER BY tier_level ASC;
```

**Users approaching next tier:**
```sql
SELECT
    user_id,
    current_tier,
    next_tier,
    tier_progress_pct,
    100 - tier_progress_pct AS pct_remaining,
    total_volume_usd,
    total_points
FROM datastreams.user_rewards FINAL
WHERE tier_progress_pct >= 80
  AND tier_level < 5  -- Not at max tier
ORDER BY tier_progress_pct DESC
LIMIT 20;
```

---

## 3. Multi-Tier Referral System

### Referral Network Structure

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     MULTI-TIER REFERRAL NETWORK                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                              ┌─────────────┐                                    │
│                              │  REFERRER   │                                    │
│                              │  (You)      │                                    │
│                              └──────┬──────┘                                    │
│                                     │                                            │
│      ┌──────────────────────────────┼──────────────────────────────┐            │
│      │                              │                              │            │
│      ▼                              ▼                              ▼            │
│   ┌──────┐                       ┌──────┐                       ┌──────┐       │
│   │  L1  │                       │  L1  │                       │  L1  │       │
│   │Direct│                       │Direct│                       │Direct│       │
│   │Ref 1 │                       │Ref 2 │                       │Ref 3 │       │
│   └──┬───┘                       └──┬───┘                       └──┬───┘       │
│      │                              │                              │            │
│      ▼                              ▼                              ▼            │
│   ┌──────┐                       ┌──────┐                       ┌──────┐       │
│   │  L2  │                       │  L2  │                       │  L2  │       │
│   │      │                       │      │                       │      │       │
│   └──┬───┘                       └──┬───┘                       └──────┘       │
│      │                              │                                           │
│      ▼                              ▼                                           │
│   ┌──────┐                       ┌──────┐                                      │
│   │  L3  │                       │  L3  │                                      │
│   │      │                       │      │                                      │
│   └──┬───┘                       └──────┘                                      │
│      │                                                                          │
│      ▼                                                                          │
│   ┌──────┐                                                                      │
│   │  L4  │                                                                      │
│   │      │                                                                      │
│   └──────┘                                                                      │
│                                                                                  │
│   EARNINGS DISTRIBUTION:                                                        │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  Layer │ Description          │ Typical Rate │ Your Earnings           │   │
│   │────────│──────────────────────│──────────────│─────────────────────────│   │
│   │   L1   │ Direct referrals     │    40%       │ Highest share           │   │
│   │   L2   │ Referrals of L1      │    25%       │ Moderate share          │   │
│   │   L3   │ Referrals of L2      │    20%       │ Smaller share           │   │
│   │   L4   │ Referrals of L3      │    15%       │ Smallest share          │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Struct Definitions

```rust
/// Referral earnings overview - aggregates earnings FOR a referrer from their downline
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct ReferralEarningsOverview {
    /// The referrer's user ID
    pub referrer_user_id: String,

    /// Total earnings across all layers (SOL)
    pub total_earnings_sol: Decimal,
    pub total_earnings_usd: Decimal,

    /// Earnings by layer (SOL)
    pub l1_earnings_sol: Decimal,  // Direct referrals
    pub l2_earnings_sol: Decimal,  // Referrals of referrals
    pub l3_earnings_sol: Decimal,
    pub l4_earnings_sol: Decimal,

    /// Total volume from downline network
    pub total_referrals_volume_sol: Decimal,
    pub total_referrals_volume_usd: Decimal,

    /// Trade counts by layer
    pub total_trade_count: u64,
    pub l1_trade_count: u64,
    pub l2_trade_count: u64,
    pub l3_trade_count: u64,
    pub l4_trade_count: u64,

    pub last_updated: OffsetDateTime,
}

/// Individual referral earning event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReferralEarning {
    /// The referrer who earned this reward
    pub referrer_user_id: String,

    /// Referral layer (1-4)
    pub referral_layer: u8,

    /// Earning amount
    pub earning_amount_sol: Decimal,
    pub fee_share_lamports: u64,

    /// Trade that generated this earning
    pub trade_volume_sol: Decimal,
    pub trade_volume_usd: Decimal,
    pub trader_user_id: String,
    pub trader_address: String,
    pub click_total_fee: Decimal,

    pub timestamp: OffsetDateTime,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- REFERRAL EARNINGS - CURRENT STATE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Aggregate earnings per referrer across all layers
-- Engine: ReplacingMergeTree - One row per referrer
-- Updates: On every trade in the referrer's downline
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.referral_earnings ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- REFERRER IDENTIFICATION
    -- ═══════════════════════════════════════════════════════════════════════

    referrer_user_id String COMMENT 'Referrer user ID',
    last_updated DateTime64(9) COMMENT 'Last update timestamp (version)',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TOTAL EARNINGS
    -- ═══════════════════════════════════════════════════════════════════════

    total_earnings_sol Decimal128(18) COMMENT 'Total earnings from all layers',
    total_earnings_usd Decimal128(18) COMMENT 'Total earnings in USD',

    -- ═══════════════════════════════════════════════════════════════════════
    -- EARNINGS BY LAYER
    -- ═══════════════════════════════════════════════════════════════════════

    l1_earnings_sol Decimal128(18) COMMENT 'L1 (direct) earnings',
    l2_earnings_sol Decimal128(18) COMMENT 'L2 earnings',
    l3_earnings_sol Decimal128(18) COMMENT 'L3 earnings',
    l4_earnings_sol Decimal128(18) COMMENT 'L4 earnings',

    -- ═══════════════════════════════════════════════════════════════════════
    -- NETWORK VOLUME
    -- ═══════════════════════════════════════════════════════════════════════

    total_referrals_volume_sol Decimal128(18) COMMENT 'Total volume from downline',
    total_referrals_volume_usd Decimal128(18) COMMENT 'Volume in USD',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TRADE COUNTS BY LAYER
    -- ═══════════════════════════════════════════════════════════════════════

    total_trade_count UInt64 COMMENT 'Total trades from downline',
    l1_trade_count UInt64 COMMENT 'L1 trade count',
    l2_trade_count UInt64 COMMENT 'L2 trade count',
    l3_trade_count UInt64 COMMENT 'L3 trade count',
    l4_trade_count UInt64 COMMENT 'L4 trade count',

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Effective referral rate
    effective_rate_pct Decimal128(18) MATERIALIZED
        if(total_referrals_volume_sol > 0,
           (total_earnings_sol / total_referrals_volume_sol) * 100, 0),

    -- Layer share percentages
    l1_share_pct Decimal128(18) MATERIALIZED
        if(total_earnings_sol > 0, (l1_earnings_sol / total_earnings_sol) * 100, 0),
    l2_share_pct Decimal128(18) MATERIALIZED
        if(total_earnings_sol > 0, (l2_earnings_sol / total_earnings_sol) * 100, 0),
    l3_share_pct Decimal128(18) MATERIALIZED
        if(total_earnings_sol > 0, (l3_earnings_sol / total_earnings_sol) * 100, 0),
    l4_share_pct Decimal128(18) MATERIALIZED
        if(total_earnings_sol > 0, (l4_earnings_sol / total_earnings_sol) * 100, 0),

    -- Active layers count
    active_layers UInt8 MATERIALIZED
        (l1_trade_count > 0)::UInt8 +
        (l2_trade_count > 0)::UInt8 +
        (l3_trade_count > 0)::UInt8 +
        (l4_trade_count > 0)::UInt8,

    -- Referrer category
    referrer_category LowCardinality(String) MATERIALIZED
        CASE
            WHEN total_earnings_sol < 1 THEN 'Starter'
            WHEN total_earnings_sol < 10 THEN 'Active'
            WHEN total_earnings_sol < 100 THEN 'Power'
            WHEN total_earnings_sol < 1000 THEN 'Super'
            ELSE 'Elite'
        END,

    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/referral_earnings',
    '{replica}',
    last_updated
)
ORDER BY (referrer_user_id)
COMMENT 'Referral earnings overview. One row per referrer.'
SETTINGS index_granularity = 8192;
```

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- REFERRAL EARNING EVENTS - INDIVIDUAL TRANSACTIONS
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Detailed log of every referral earning event
-- Engine: MergeTree - Append-only event log
-- Use Case: "Which trades generated earnings for me?"
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.referral_earning_events ON CLUSTER `server-apex`
(
    -- Referrer info
    referrer_user_id String COMMENT 'Referrer who earned',
    referral_layer UInt8 COMMENT 'Layer 1-4',

    -- Earning details
    earning_amount_sol Decimal128(18) COMMENT 'Earning in SOL',
    fee_share_lamports UInt64 COMMENT 'Earning in lamports (precision)',

    -- Trade that generated this
    trade_volume_sol Decimal128(18) COMMENT 'Trade volume that generated earning',
    trade_volume_usd Decimal128(18) COMMENT 'Trade volume in USD',
    trader_user_id String COMMENT 'Trader who generated earning',
    trader_address String COMMENT 'Trader wallet',
    click_total_fee Decimal128(18) COMMENT 'Click fee from trade',

    -- Timestamps
    timestamp DateTime64(9) COMMENT 'Earning timestamp',
    event_date Date MATERIALIZED toDate(timestamp),
    ingested_at DateTime64(3) DEFAULT now64(3),

    -- Indexes
    INDEX idx_trader trader_user_id TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/referral_earning_events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (referrer_user_id, timestamp)
TTL event_date + INTERVAL 1 YEAR
COMMENT 'Individual referral earning events. TTL: 1 year.'
SETTINGS index_granularity = 8192;
```

### Referral Analytics Queries

**Referrer's earnings summary:**
```sql
SELECT
    referrer_user_id,
    referrer_category,
    total_earnings_sol,
    total_earnings_usd,
    l1_earnings_sol,
    l2_earnings_sol,
    l3_earnings_sol,
    l4_earnings_sol,
    l1_share_pct,
    active_layers,
    total_trade_count,
    effective_rate_pct
FROM datastreams.referral_earnings FINAL
WHERE referrer_user_id = {referrer_id:String};
```

**Top referrers leaderboard:**
```sql
SELECT
    referrer_user_id,
    referrer_category,
    total_earnings_sol,
    total_earnings_usd,
    active_layers,
    l1_share_pct,
    total_trade_count,
    total_referrals_volume_usd,
    effective_rate_pct
FROM datastreams.referral_earnings FINAL
ORDER BY total_earnings_sol DESC
LIMIT 100;
```

**Recent earning events for referrer:**
```sql
SELECT
    referral_layer,
    earning_amount_sol,
    trade_volume_sol,
    trade_volume_usd,
    trader_user_id,
    timestamp
FROM datastreams.referral_earning_events
WHERE referrer_user_id = {referrer_id:String}
ORDER BY timestamp DESC
LIMIT 50;
```

**Earnings by layer over time:**
```sql
SELECT
    toDate(timestamp) AS day,
    sumIf(earning_amount_sol, referral_layer = 1) AS l1_earnings,
    sumIf(earning_amount_sol, referral_layer = 2) AS l2_earnings,
    sumIf(earning_amount_sol, referral_layer = 3) AS l3_earnings,
    sumIf(earning_amount_sol, referral_layer = 4) AS l4_earnings,
    sum(earning_amount_sol) AS total_earnings,
    count() AS trade_count,
    uniq(trader_user_id) AS unique_traders
FROM datastreams.referral_earning_events
WHERE referrer_user_id = {referrer_id:String}
  AND timestamp >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day;
```

**Network layer performance analysis:**
```sql
SELECT
    referral_layer,
    count() AS earning_events,
    sum(earning_amount_sol) AS total_earnings,
    avg(earning_amount_sol) AS avg_earning_per_event,
    sum(trade_volume_sol) AS total_volume,
    uniq(referrer_user_id) AS unique_referrers,
    uniq(trader_user_id) AS unique_traders
FROM datastreams.referral_earning_events
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY referral_layer
ORDER BY referral_layer;
```

**Top earning referrer-trader pairs:**
```sql
SELECT
    referrer_user_id,
    trader_user_id,
    count() AS trade_count,
    sum(earning_amount_sol) AS total_earned,
    sum(trade_volume_sol) AS trader_volume,
    min(timestamp) AS first_trade,
    max(timestamp) AS last_trade
FROM datastreams.referral_earning_events
WHERE referrer_user_id = {referrer_id:String}
GROUP BY referrer_user_id, trader_user_id
ORDER BY total_earned DESC
LIMIT 20;
```

---

## 4. Aggregation Tables for Dashboards

### Daily User Activity Summary

```sql
-- Pre-aggregated daily stats for fast dashboard loading
CREATE TABLE datastreams.user_daily_summary ON CLUSTER `server-apex`
(
    user_id String,
    day Date,

    -- Order metrics
    total_orders UInt64,
    market_orders UInt64,
    limit_orders UInt64,
    dca_orders UInt64,
    filled_orders UInt64,
    cancelled_orders UInt64,
    order_volume_usd Decimal128(18),

    -- Reward metrics
    points_earned Decimal128(18),
    cashback_earned Decimal128(18),

    -- Referral metrics
    referral_earnings Decimal128(18),
    referral_trades UInt64
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (user_id, day)
COMMENT 'Pre-aggregated daily user activity for dashboards.';
```

### Platform-Wide Daily Metrics

```sql
-- Platform health metrics
CREATE TABLE datastreams.platform_daily_metrics ON CLUSTER `server-apex`
(
    day Date,

    -- User metrics
    active_users UInt64,
    new_users UInt64,

    -- Order metrics
    total_orders UInt64,
    total_order_volume_usd Decimal128(18),

    -- Reward metrics
    points_distributed Decimal128(18),
    cashback_distributed Decimal128(18),
    cashback_claimed Decimal128(18),

    -- Referral metrics
    referral_earnings_distributed Decimal128(18),
    active_referrers UInt64
)
ENGINE = SummingMergeTree()
ORDER BY day
COMMENT 'Platform-wide daily health metrics.';
```

---

## Summary

This concludes the 9-part series on building real-time Solana DEX analytics with ClickHouse. In this final part, we covered Click platform-specific features:

| Feature | Current State Table | History/Events Table | Key Metrics |
|---------|---------------------|---------------------|-------------|
| **Orders** | `order_overview` | `order_history` | Fill %, VWAP, trade count |
| **Rewards** | `user_rewards` | `user_rewards_history` | Points, tier, cashback |
| **Referrals** | `referral_earnings` | `referral_earning_events` | L1-L4 breakdown, network depth |

### Key Patterns Used Throughout the Series

1. **ReplacingMergeTree** for current state (positions, accounts, orders)
2. **MergeTree** for event history (trades, funding, earnings)
3. **FINAL** modifier for consistent reads from ReplacingMergeTree
4. **Materialized columns** for computed fields (risk scores, percentages)
5. **Bloom filter indexes** for high-cardinality lookups
6. **Tiered storage** for cost-effective data retention
7. **Partitioning by month** for efficient time-range queries

### Complete Blog Series

1. **Architecture Overview** - System design and data flow
2. **Trade Enrichment & OHLCV** - Real-time price aggregation
3. **Token Metrics & Rolling Windows** - Statistical calculations
4. **Position Tracking & P&L** - Trader portfolio management
5. **Discovery & Trending Analytics** - Token lifecycle and leaderboards
6. **Production Patterns** - Optimization and monitoring
7. **Hyperliquid Perps** - Derivatives trading analytics
8. **Risk Analysis & External Data** - Security and wallet classification
9. **Platform Features** - Orders, rewards, and referrals

This architecture processes millions of events daily, maintaining sub-second query latency while storing months of historical data efficiently.
