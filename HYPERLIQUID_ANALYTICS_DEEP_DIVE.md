# The Complete Guide to Hyperliquid DEX Trading Analytics: Know Every Number, Master Every Trade

## Introduction: Institutional-Grade Analytics for Every Trader

In traditional finance, institutional traders have access to sophisticated analytics platforms costing thousands of dollars per month. Bloomberg terminals, proprietary risk management systems, performance tracking software—the tools that separate professional traders from amateurs.

We've built those same institutional-grade analytics for Hyperliquid DEX traders. For free.

Whether you're trading with $100 or $10 million, you get access to the same deep performance metrics, risk analytics, competitive leaderboards, and historical tracking that professional trading desks use to manage billions.

---

## The Theory Behind Trading Analytics

### Why Data-Driven Trading Matters

Trading without analytics is like flying blind. Research from behavioral finance shows that traders who track their performance improve their results by 20-40% over time. Here's why:

**1. The Feedback Loop Problem**
Most traders suffer from confirmation bias—they remember their wins and forget their losses. Without objective data, you can't accurately assess your trading edge. Our analytics provide the unbiased truth.

**2. Pattern Recognition**
Markets are complex adaptive systems, but your own trading patterns are not. By analyzing thousands of your trades, we can identify:
- Time-of-day effects (when you trade best/worst)
- Market conditions where you excel
- Position sizes that lead to emotional decisions
- Recovery patterns after drawdowns

**3. The Kelly Criterion and Position Sizing**
Optimal position sizing depends on your edge (win rate × avg win - loss rate × avg loss). Without accurate tracking, you can't calculate your edge—and you're either overleveraged or leaving money on the table.

### The Mathematics of Trading Performance

**Expected Value (EV):**
```
EV = (Win Rate × Average Win) - (Loss Rate × Average Loss)
```
A positive EV means you have an edge. Our analytics calculate this automatically across all your trades.

**Risk-Adjusted Returns:**
Raw returns don't tell the full story. A trader who makes 100% with 80% drawdown is taking excessive risk compared to one making 50% with 20% drawdown. That's why we calculate Sharpe and Sortino ratios.

---

### The Data Behind the Analytics

Our analytics platform processes:

| Metric | Scale |
|--------|-------|
| **Total Database Size** | 2.5+ TB |
| **Hypertables** | 57 time-series tables |
| **Continuous Aggregates** | 131 pre-computed rollups |
| **Materialized Views** | 16 complex analytics views |
| **Trade Records** | 1+ billion fills tracked |
| **Wallet Snapshots** | 560M+ leaderboard entries |
| **PnL History Points** | 16M+ snapshots |

This isn't sample data or estimates. Every single trade, every position change, every liquidation—captured and analyzed in real-time.

---

## Part 1: Performance Metrics - Your Trading Report Card

### Understanding Your Win Rate

**What It Is:**
Your win rate is the percentage of your trades that close in profit. If you execute 100 trades and 60 are profitable, your win rate is 60%.

**How We Calculate It:**
```sql
-- Win Rate Calculation from hl_user_fills
SELECT
    address,
    COUNT(*) FILTER (WHERE closedPnl > 0) as winning_trades,
    COUNT(*) FILTER (WHERE closedPnl < 0) as losing_trades,
    COUNT(*) as total_trades,
    ROUND(100.0 * COUNT(*) FILTER (WHERE closedPnl > 0) / NULLIF(COUNT(*), 0), 2) as win_rate_pct
FROM hl_user_fills
WHERE closedPnl IS NOT NULL
GROUP BY address;
```

We track every single trade you execute, categorize each one as profitable or unprofitable, and calculate your win rate across different dimensions:

- **All-time win rate** - Your career performance
- **Monthly win rate** - Recent performance trends
- **Weekly win rate** - Short-term execution quality
- **Win rate by market** - Which coins you're best at trading (BTC, ETH, SOL, etc.)
- **Win rate by direction** - Are you better at longs or shorts?

**Why It Matters:**

Many traders assume they need a high win rate to be profitable. Actually, professional traders often have win rates of 40-50%. What matters more is whether your winning trades are bigger than your losing trades.

**Example Scenarios:**

*Trader A: High Win Rate, Low Profitability*
- Win Rate: 70%
- Average Win: $50
- Average Loss: $200
- Result: Losing money despite high win rate

*Trader B: Lower Win Rate, High Profitability*
- Win Rate: 45%
- Average Win: $500
- Average Loss: $100
- Result: Highly profitable with "losing" win rate

**Win Rate Benchmarks:**
| Range | Assessment |
|-------|------------|
| Below 35% | Red flag. Strategy needs serious revision. |
| 35-45% | Acceptable if wins are 2:1 larger than losses. |
| 45-55% | Good balance. Aim for at least 1:1 win/loss ratio. |
| 55-65% | Strong performance. Even with 1:1 ratio, you're profitable. |
| Above 65% | Exceptional. Make sure you're not cutting winners too early. |

---

### Realized Profit & Loss (PnL) - The Truth About Your Trading

**What It Is:**
Realized PnL is the actual money you've made or lost from trades that you've closed. This is real profit and loss that has hit your account—not theoretical, not potential, but actual.

**How We Calculate It:**

```sql
-- Realized PnL from closed positions
SELECT
    address,
    token,
    SUM(closedPnl) as realized_pnl,
    COUNT(*) as trade_count,
    AVG(closedPnl) as avg_pnl_per_trade
FROM hl_user_fills
WHERE closedPnl IS NOT NULL
GROUP BY address, token
ORDER BY realized_pnl DESC;
```

**We Track:**
- **Total Realized PnL:** Your cumulative profit/loss from all closed trades ever
- **Daily Realized PnL:** How much you made/lost today
- **Weekly/Monthly/Yearly Realized PnL:** Performance over different time periods
- **Realized PnL by Market:** Which coins are profitable for you
- **Realized PnL by Direction:** Are you making money on longs, shorts, or both?

**Real Example from Our Data:**

One wallet on our platform shows:
- Realized PnL: -$4,383,527 (actual loss)
- Unrealized PnL: +$43,357,503 (paper gains on open positions)

Which number matters? Both, but the realized number tells you what actually happened. This trader lost $4.3 million in actual trading, even though they're currently sitting on paper gains.

**How to Interpret:**
| Scenario | What It Means |
|----------|---------------|
| Positive Realized PnL | You're extracting real profits from the market. Well done. |
| Negative Realized, Positive Unrealized | You're holding bags hoping for recovery. Be honest. |
| Positive Realized, Negative Unrealized | You take profits and let some losers run. Common pattern. |
| Both Negative | Reevaluate your strategy immediately. |

---

### Unrealized Profit & Loss - What Could Be

**What It Is:**
Unrealized PnL shows the current profit or loss on your open positions. If you closed them right now at market price, this is what you'd gain or lose.

**How We Calculate It:**

```sql
-- Current unrealized PnL from open positions
SELECT
    p.address,
    p.token,
    p.size,
    p.entry_price,
    pt.mark_price as current_price,
    (pt.mark_price - p.entry_price) * p.size as unrealized_pnl
FROM perp_asset_positions p
JOIN perp_tokens pt ON p.token = pt.token
WHERE p.size != 0
ORDER BY ABS((pt.mark_price - p.entry_price) * p.size) DESC;
```

**Why It Matters:**
1. **Current Portfolio Health:** Are your open positions winning or losing right now?
2. **Risk Exposure:** Large unrealized losses indicate positions that might need attention
3. **Opportunity Cost:** Large unrealized gains might be worth taking profits on
4. **Emotional Check:** If you can't handle the unrealized loss, your position is too big

**Different Perspectives:**

*Conservative Trader View:*
"Unrealized gains aren't real until I close the position. I focus on realized PnL."

*Optimistic Trader View:*
"My total profitability includes unrealized gains. My strategy is working."

*Professional Trader View:*
"I track both. Realized tells me my execution skill. Unrealized tells me my current risk."

---

### Sharpe Ratio - Risk-Adjusted Returns

**What It Is:**
The Sharpe Ratio is a sophisticated metric that measures how much return you're getting for the risk you're taking. It's the metric that professional fund managers use to evaluate performance.

**How We Calculate It:**

```sql
-- Sharpe Ratio calculation using continuous aggregate data
WITH daily_returns AS (
    SELECT
        address,
        DATE_TRUNC('day', "timestampUTC") as day,
        SUM(closedPnl) as daily_pnl
    FROM hl_user_fills
    WHERE closedPnl IS NOT NULL
    GROUP BY address, DATE_TRUNC('day', "timestampUTC")
),
stats AS (
    SELECT
        address,
        AVG(daily_pnl) as mean_daily_pnl,
        STDDEV(daily_pnl) as stddev_daily_pnl
    FROM daily_returns
    GROUP BY address
    HAVING COUNT(*) >= 30  -- Minimum 30 days of data
)
SELECT
    address,
    mean_daily_pnl,
    stddev_daily_pnl,
    CASE
        WHEN stddev_daily_pnl > 0
        THEN ROUND((mean_daily_pnl / stddev_daily_pnl) * SQRT(252), 3)  -- Annualized
        ELSE NULL
    END as sharpe_ratio
FROM stats
ORDER BY sharpe_ratio DESC NULLS LAST;
```

**Understanding the Formula:**
```
Sharpe Ratio = Mean PnL Per Trade / Standard Deviation of PnL
```

In simpler terms: Your average profit divided by how volatile your profits are.

**Sharpe Ratio Benchmarks:**
| Sharpe Ratio | Assessment |
|--------------|------------|
| Below 0 | Losing money on average. No edge. |
| 0 to 0.5 | Marginal edge with high volatility. Risky. |
| 0.5 to 1.0 | Decent edge with moderate risk. Acceptable. |
| 1.0 to 2.0 | Strong edge with controlled risk. Good trading. |
| 2.0 to 3.0 | Excellent edge with low risk. Professional level. |
| Above 3.0 | Exceptional. Either amazing skill or not enough data yet. |

**Real-World Application:**
Two traders both made $100,000 last year:

*Trader A:*
- Made $100k through 1,000 trades averaging $100 each
- Standard deviation: $50
- Sharpe Ratio: 2.0
- **Experience:** Steady, consistent, low stress

*Trader B:*
- Made $100k through wild swings, huge wins and losses
- Standard deviation: $500
- Sharpe Ratio: 0.2
- **Experience:** Stressful, emotional rollercoaster, unsustainable

Who would you rather be?

---

### Sortino Ratio - Downside-Adjusted Returns

**What It Is:**
The Sortino Ratio is similar to the Sharpe Ratio but only considers downside volatility. It doesn't penalize you for upside volatility (big wins).

**How We Calculate It:**

```sql
-- Sortino Ratio calculation
WITH daily_returns AS (
    SELECT
        address,
        DATE_TRUNC('day', "timestampUTC") as day,
        SUM(closedPnl) as daily_pnl
    FROM hl_user_fills
    WHERE closedPnl IS NOT NULL
    GROUP BY address, DATE_TRUNC('day', "timestampUTC")
),
stats AS (
    SELECT
        address,
        AVG(daily_pnl) as mean_daily_pnl,
        STDDEV(daily_pnl) FILTER (WHERE daily_pnl < 0) as downside_stddev
    FROM daily_returns
    GROUP BY address
    HAVING COUNT(*) >= 30
)
SELECT
    address,
    mean_daily_pnl,
    downside_stddev,
    CASE
        WHEN downside_stddev > 0
        THEN ROUND((mean_daily_pnl / downside_stddev) * SQRT(252), 3)
        ELSE NULL
    END as sortino_ratio
FROM stats
ORDER BY sortino_ratio DESC NULLS LAST;
```

**Why Sortino is Sometimes Better:**
If you have a home run strategy with occasional big wins, the Sharpe Ratio penalizes that volatility even though it's positive volatility. The Sortino Ratio only cares about your losing days.

---

### Maximum Drawdown - Your Worst Period

**What It Is:**
Maximum drawdown measures the largest peak-to-trough decline in your account. It tells you: "What's the worst it ever got?"

**How We Calculate It:**

```sql
-- Maximum Drawdown calculation
WITH cumulative_pnl AS (
    SELECT
        address,
        "timestampUTC",
        SUM(closedPnl) OVER (PARTITION BY address ORDER BY "timestampUTC") as running_pnl
    FROM hl_user_fills
    WHERE closedPnl IS NOT NULL
),
running_max AS (
    SELECT
        address,
        "timestampUTC",
        running_pnl,
        MAX(running_pnl) OVER (PARTITION BY address ORDER BY "timestampUTC") as peak_pnl
    FROM cumulative_pnl
),
drawdowns AS (
    SELECT
        address,
        "timestampUTC",
        running_pnl,
        peak_pnl,
        running_pnl - peak_pnl as drawdown,
        CASE WHEN peak_pnl > 0
             THEN (running_pnl - peak_pnl) / peak_pnl * 100
             ELSE NULL
        END as drawdown_pct
    FROM running_max
)
SELECT
    address,
    MIN(drawdown) as max_drawdown_usd,
    MIN(drawdown_pct) as max_drawdown_pct
FROM drawdowns
GROUP BY address
ORDER BY max_drawdown_usd;
```

**Why It Matters:**
- **Risk Management:** Shows how much pain you've endured
- **Recovery Tracking:** How far do you need to climb back?
- **Position Sizing:** If your max drawdown exceeds your risk tolerance, reduce size
- **Strategy Validation:** Big drawdowns might indicate strategy flaws

**Drawdown Psychology:**

| Drawdown | Gain Needed to Recover | Psychological Impact |
|----------|----------------------|---------------------|
| -10% | +11% | Annoying but manageable |
| -20% | +25% | Concerning, time to reassess |
| -30% | +43% | Serious, major strategy review needed |
| -50% | +100% | Critical, need to double your money |
| -75% | +300% | Devastating, career-threatening |

---

## Part 2: The 23-Tier Leaderboard System

### Overview: Trading as a Competitive Sport

We've transformed trading from a solitary activity into a competitive, gamified experience. Our leaderboard system ranks hundreds of thousands of traders across multiple dimensions.

**Two Independent Ranking Systems:**
1. **Value Rank:** Based on total portfolio value (trading + staking)
2. **VLM Rank:** Based on lifetime trading volume

You can be a whale by value but rank lower in VLM, or vice versa. Both matter.

### The Complete Tier Hierarchy

#### Elite Tiers: The Top 0.1%

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Blue Whale I** | $50,000,000+ | The absolute pinnacle. Fewer than 10 wallets globally. |
| **Blue Whale II** | $25,000,000 - $50,000,000 | Multi-generational wealth territory. |
| **Blue Whale III** | $10,000,000 - $25,000,000 | Ultra-high net worth traders. |

#### Large Whale Tiers: The Top 1%

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Large Whale I** | $6,000,000 - $10,000,000 | Small family office level capital. |
| **Large Whale II** | $3,500,000 - $6,000,000 | Independently wealthy from trading. |
| **Large Whale III** | $2,000,000 - $3,500,000 | "Never work again" threshold. |

#### Whale Tiers: Top 5%

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Whale I** | $1,200,000 - $2,000,000 | Millionaire status achieved. |
| **Whale II** | $800,000 - $1,200,000 | Approaching millionaire status. |
| **Whale III** | $500,000 - $800,000 | Half-million club. |

#### Small Whale Tiers: Top 10-15%

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Small Whale I** | $300,000 - $500,000 | Solid six-figure portfolio. |
| **Small Whale II** | $175,000 - $300,000 | Building toward major milestones. |
| **Small Whale III** | $100,000 - $175,000 | Six-figure milestone achieved. |

#### Shark Tiers: Top 20-30%

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Shark I** | $60,000 - $100,000 | Strong five-figure portfolio. |
| **Shark II** | $30,000 - $60,000 | Mid five-figure portfolio. |
| **Shark III** | $10,000 - $30,000 | Breaking into meaningful capital. |

#### Dolphin Tiers: Active Traders (30-50%)

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Dolphin I** | $7,000 - $10,000 | Solid four-figure portfolio. |
| **Dolphin II** | $4,500 - $7,000 | Mid four-figure portfolio. |
| **Dolphin III** | $2,500 - $4,500 | Entry-level serious trading. |

#### Barracuda Tiers: Growing Traders (50-70%)

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Barracuda I** | $1,750 - $2,500 | Small but meaningful portfolio. |
| **Barracuda II** | $1,000 - $1,750 | Four-figure portfolio achieved. |
| **Barracuda III** | $500 - $1,000 | First serious capital deployment. |

#### Minnow Tiers: Beginners (70-100%)

| Tier | Value Range | Description |
|------|-------------|-------------|
| **Minnow I** | $250 - $500 | Small starting capital. |
| **Minnow II** | $100 - $250 | Minimal capital. |
| **Minnow III** | Under $100 | Starting position. Everyone begins here. |

### Tier Progression Tracking

**SQL Query for Tier History:**

```sql
-- Track tier progression over time
SELECT
    address,
    DATE_TRUNC('day', created_at) as snapshot_date,
    total_value,
    tier,
    LAG(tier) OVER (PARTITION BY address ORDER BY created_at) as previous_tier,
    CASE
        WHEN tier != LAG(tier) OVER (PARTITION BY address ORDER BY created_at)
        THEN 'TIER_CHANGE'
        ELSE 'STABLE'
    END as tier_status
FROM leaderboard
WHERE address = '0x...'
ORDER BY created_at DESC;
```

---

## Part 3: PnL Achievement System - The 8 Performance Badges

Beyond total value, we recognize trading skill through PnL-based achievements. These badges are pure skill recognition—they're about how well you trade, not how much capital you started with.

### Elite Performers

| Badge | PnL Range | Rarity | Description |
|-------|-----------|--------|-------------|
| **Money Printer** | $1,000,000+ | <0.01% | Seven figures extracted from the market. Elite trading skill. |
| **Smart Money** | $100,000 - $1,000,000 | <0.1% | Six-figure profits. Consistently profitable and skilled. |
| **Grinder** | $10,000 - $100,000 | <1% | Five-figure profits. Proven edge in the market. |
| **Humble Earner** | $0 - $10,000 | ~5-15% | Profitable trader. You have an edge (most traders don't). |

### The Learning Curve

| Badge | PnL Range | Description |
|-------|-----------|-------------|
| **Exit Liquidity** | -$10,000 to $0 | Small losses. Learning expensive lessons but recoverable. |
| **Semi-Rekt** | -$100,000 to -$10,000 | Significant five-figure losses. Strategy revision needed. |
| **Full Rekt** | -$1,000,000 to -$100,000 | Six-figure losses. Catastrophic strategy failure. |
| **Giga Rekt** | Below -$1,000,000 | Seven-figure losses. Maximum damage. |

### Badge Distribution Query

```sql
-- Calculate badge distribution across all wallets
WITH wallet_pnl AS (
    SELECT
        address,
        SUM(closedPnl) as total_realized_pnl
    FROM hl_user_fills
    WHERE closedPnl IS NOT NULL
    GROUP BY address
),
badges AS (
    SELECT
        address,
        total_realized_pnl,
        CASE
            WHEN total_realized_pnl >= 1000000 THEN 'Money Printer'
            WHEN total_realized_pnl >= 100000 THEN 'Smart Money'
            WHEN total_realized_pnl >= 10000 THEN 'Grinder'
            WHEN total_realized_pnl >= 0 THEN 'Humble Earner'
            WHEN total_realized_pnl >= -10000 THEN 'Exit Liquidity'
            WHEN total_realized_pnl >= -100000 THEN 'Semi-Rekt'
            WHEN total_realized_pnl >= -1000000 THEN 'Full Rekt'
            ELSE 'Giga Rekt'
        END as badge
    FROM wallet_pnl
)
SELECT
    badge,
    COUNT(*) as wallet_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM badges
GROUP BY badge
ORDER BY
    CASE badge
        WHEN 'Money Printer' THEN 1
        WHEN 'Smart Money' THEN 2
        WHEN 'Grinder' THEN 3
        WHEN 'Humble Earner' THEN 4
        WHEN 'Exit Liquidity' THEN 5
        WHEN 'Semi-Rekt' THEN 6
        WHEN 'Full Rekt' THEN 7
        WHEN 'Giga Rekt' THEN 8
    END;
```

---

## Part 4: Trading Activity Insights

### What We Track

Our `hl_user_fills` table contains over **1 billion trade records** with the following key fields:

| Field | Description |
|-------|-------------|
| `address` | Trader wallet address |
| `token` | Trading pair (BTC, ETH, SOL, etc.) |
| `side` | Buy or Sell |
| `size` | Position size |
| `price` | Execution price |
| `closedPnl` | Realized PnL for this fill |
| `timestampUTC` | Exact execution timestamp |
| `fee` | Trading fee paid |

### Trading Volume Analysis

```sql
-- Trading volume by market
SELECT
    token,
    COUNT(*) as total_trades,
    SUM(size * price) as total_volume,
    AVG(size * price) as avg_trade_size,
    SUM(closedPnl) as total_pnl
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY token
ORDER BY total_volume DESC
LIMIT 20;
```

### Long/Short Analysis

```sql
-- Long vs Short performance by wallet
SELECT
    address,
    side,
    COUNT(*) as trade_count,
    SUM(closedPnl) as total_pnl,
    AVG(closedPnl) as avg_pnl,
    COUNT(*) FILTER (WHERE closedPnl > 0) as winning_trades,
    ROUND(100.0 * COUNT(*) FILTER (WHERE closedPnl > 0) / NULLIF(COUNT(*), 0), 2) as win_rate
FROM hl_user_fills
WHERE closedPnl IS NOT NULL
GROUP BY address, side
ORDER BY address, side;
```

### Activity Patterns

**Trading by Time of Day:**
```sql
-- Trading activity by hour (UTC)
SELECT
    EXTRACT(HOUR FROM "timestampUTC") as hour_utc,
    COUNT(*) as trade_count,
    SUM(size * price) as volume,
    AVG(closedPnl) as avg_pnl
FROM hl_user_fills
GROUP BY EXTRACT(HOUR FROM "timestampUTC")
ORDER BY hour_utc;
```

**Trading by Day of Week:**
```sql
-- Trading activity by day of week
SELECT
    EXTRACT(DOW FROM "timestampUTC") as day_of_week,
    TO_CHAR("timestampUTC", 'Day') as day_name,
    COUNT(*) as trade_count,
    SUM(closedPnl) as total_pnl,
    AVG(closedPnl) as avg_pnl
FROM hl_user_fills
WHERE closedPnl IS NOT NULL
GROUP BY EXTRACT(DOW FROM "timestampUTC"), TO_CHAR("timestampUTC", 'Day')
ORDER BY day_of_week;
```

---

## Part 5: Liquidation Tracking & Analysis

### Data Source: `hl_liquidation_orders`

Every liquidation event is captured with full details:

| Field | Description |
|-------|-------------|
| `address` | Liquidated wallet |
| `token` | Market where liquidation occurred |
| `side` | Long or Short position |
| `size` | Position size liquidated |
| `liquidation_price` | Price at liquidation |
| `entry_price` | Original entry price |
| `leverage` | Leverage used |
| `timestamp` | Exact liquidation time |

### Personal Liquidation Analysis

```sql
-- Your liquidation history
SELECT
    token,
    side,
    size,
    entry_price,
    liquidation_price,
    ABS(liquidation_price - entry_price) * size as loss_amount,
    leverage,
    timestamp
FROM hl_liquidation_orders
WHERE address = '0x...'
ORDER BY timestamp DESC;
```

### Platform-Wide Liquidation Metrics

```sql
-- Liquidation summary for last 24 hours
SELECT
    DATE_TRUNC('hour', timestamp) as hour,
    SUM(CASE WHEN side = 'long' THEN size * liquidation_price ELSE 0 END) as long_liquidations,
    SUM(CASE WHEN side = 'short' THEN size * liquidation_price ELSE 0 END) as short_liquidations,
    COUNT(*) as liquidation_count
FROM hl_liquidation_orders
WHERE timestamp >= NOW() - INTERVAL '24 hours'
GROUP BY DATE_TRUNC('hour', timestamp)
ORDER BY hour DESC;
```

### Liquidation Heatmap Data

Our `liquidation_histogram_30min` materialized view provides pre-computed heatmap data showing:

- Liquidation concentration by price level
- Time-of-day patterns
- Leverage distribution at liquidation
- Recovery rates after liquidation

---

## Part 6: The Power of Continuous Aggregates

### What Are Continuous Aggregates?

TimescaleDB Continuous Aggregates are **pre-computed, incrementally updated materialized views** that enable sub-second query performance on terabytes of data.

We have **131 continuous aggregates** powering real-time dashboards:

### Key Continuous Aggregates

| CAGG Name | Purpose | Refresh Interval |
|-----------|---------|------------------|
| `leaderboard_30min` | Real-time rankings | 30 minutes |
| `pnl_daily` | Daily PnL snapshots | 1 hour |
| `volume_by_market_hourly` | Market volume tracking | 1 hour |
| `active_traders_daily` | Active trader counts | Daily |
| `liquidation_histogram_30min` | Liquidation heatmaps | 30 minutes |
| `trader_metrics_daily` | Per-trader performance | Daily |

### Performance Comparison

| Query Type | Without CAGG | With CAGG |
|------------|--------------|-----------|
| Daily volume by market | 45 seconds | 50ms |
| Top 100 traders by PnL | 2 minutes | 100ms |
| 30-day liquidation summary | 30 seconds | 25ms |
| Win rate by trader | 1 minute | 75ms |

### How We Maintain Data Freshness

```sql
-- Refresh policy for continuous aggregates
SELECT add_continuous_aggregate_policy('leaderboard_30min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '30 minutes'
);
```

---

## Part 7: Data Architecture Deep-Dive

### Understanding Time-Series Databases

Traditional relational databases like MySQL or PostgreSQL struggle with time-series data at scale. When you have billions of rows being inserted chronologically, you face several challenges:

1. **Index bloat** - B-tree indexes grow enormous and slow down inserts
2. **Table scans** - Queries spanning time ranges become painfully slow
3. **Storage inefficiency** - No automatic data lifecycle management
4. **Query complexity** - Aggregating over time requires complex SQL

**TimescaleDB solves these problems** by extending PostgreSQL with:
- **Hypertables** - Automatic time-based partitioning
- **Continuous Aggregates** - Pre-computed rollups that auto-update
- **Native Compression** - 90%+ storage reduction with query acceleration
- **Data lifecycle policies** - Automatic retention and tiering

### Our Hypertable Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      HYPERTABLE: hl_user_fills              │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ Chunk 1 │ │ Chunk 2 │ │ Chunk 3 │ │ Chunk N │  ...     │
│  │ Week 1  │ │ Week 2  │ │ Week 3  │ │ Week N  │          │
│  │ 15M rows│ │ 18M rows│ │ 12M rows│ │  ...    │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
│                                                             │
│  Partition Key: timestampUTC                                │
│  Chunk Interval: 7 days                                     │
│  Total Chunks: 150+                                         │
│  Total Rows: 1,000,000,000+                                │
└─────────────────────────────────────────────────────────────┘
```

**Key Hypertables in Our System:**

| Hypertable | Time Column | Chunk Interval | Purpose |
|------------|-------------|----------------|---------|
| `hl_user_fills` | timestampUTC | 7 days | All trade executions |
| `hl_orderbook` | timestampUTC | 7 days | Order book snapshots |
| `perp_pnl_histories` | timestamp | 7 days | PnL tracking over time |
| `leaderboard` | created_at | 7 days | Ranking snapshots |
| `spot_tokens` | timestampUTC | 7 days | Spot token price history |
| `perp_tokens` | timestampUTC | 7 days | Perp token price history |
| `heatmaps` | timestampUTC | 7 days | Trading activity heatmaps |

### Schema Design Principles

**1. Denormalization for Query Speed**
We store wallet addresses directly in each record rather than using foreign keys. This enables faster queries without JOINs:

```sql
-- Fast: Direct access without JOIN
SELECT address, SUM(closedPnl) FROM hl_user_fills
WHERE token = 'BTC' GROUP BY address;

-- Slow: Normalized approach requiring JOIN
SELECT w.address, SUM(f.closedPnl) FROM fills f
JOIN wallets w ON f.wallet_id = w.id
WHERE f.token = 'BTC' GROUP BY w.address;
```

**2. Segment By for Compression**
We configure compression to segment by high-cardinality columns:

```sql
ALTER TABLE hl_user_fills SET (
    timescaledb.compress_segmentby = 'token',
    timescaledb.compress_orderby = 'address ASC, "timestampUTC" DESC'
);
```

This means data for each token is compressed together, enabling:
- Better compression ratios (similar data compresses better)
- Fast queries filtering by token (skip irrelevant segments)

**3. Continuous Aggregate Hierarchy**

```
Raw Data (hl_user_fills)
    │
    ├── volume_hourly (CAGG, refreshed every hour)
    │       │
    │       └── volume_daily (CAGG on CAGG, refreshed daily)
    │
    ├── pnl_hourly (CAGG)
    │       │
    │       └── pnl_daily (CAGG on CAGG)
    │
    └── leaderboard_30min (CAGG for real-time rankings)
```

---

## Part 8: API Performance Benchmarks

### Query Response Times

We've optimized our queries to deliver institutional-grade performance. Here are actual benchmarks from our production system:

| Query Type | Without Optimization | With CAGG | Improvement |
|------------|---------------------|-----------|-------------|
| Top 100 traders by PnL (30 days) | 45s | 85ms | **529x faster** |
| Daily volume by market | 38s | 42ms | **905x faster** |
| Win rate calculation (single wallet) | 12s | 120ms | **100x faster** |
| Leaderboard rankings | 2min 30s | 65ms | **2300x faster** |
| Liquidation histogram | 28s | 35ms | **800x faster** |

### How We Achieve These Numbers

**1. Pre-computed Continuous Aggregates**
Instead of scanning 1 billion rows, we query pre-computed results:

```sql
-- Without CAGG: Scans entire hl_user_fills table (45+ seconds)
SELECT address, SUM(closedPnl), COUNT(*)
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY address
ORDER BY SUM(closedPnl) DESC LIMIT 100;

-- With CAGG: Reads from pre-computed leaderboard_30min (85ms)
SELECT address, total_pnl, trade_count
FROM leaderboard_30min
WHERE bucket >= NOW() - INTERVAL '30 days'
ORDER BY total_pnl DESC LIMIT 100;
```

**2. Compression with Skip Scan**
Compressed chunks include min/max metadata that enables skipping irrelevant data:

```sql
EXPLAIN ANALYZE
SELECT * FROM hl_user_fills
WHERE token = 'BTC' AND "timestampUTC" >= '2024-01-01';

-- Output shows: "Chunks skipped: 87" (older chunks not scanned)
```

**3. Proper Indexing Strategy**

```sql
-- Composite index for common query patterns
CREATE INDEX idx_fills_addr_token_time
ON hl_user_fills (address, token, "timestampUTC" DESC);

-- Partial index for profitable trades only
CREATE INDEX idx_fills_winners
ON hl_user_fills (address, "timestampUTC")
WHERE closedPnl > 0;
```

### Latency SLAs

| Endpoint | P50 | P95 | P99 |
|----------|-----|-----|-----|
| Leaderboard | 45ms | 120ms | 250ms |
| Wallet analytics | 80ms | 200ms | 400ms |
| Market volume | 35ms | 95ms | 180ms |
| PnL history | 60ms | 150ms | 300ms |

---

## Part 9: Data Freshness Guarantees

### Refresh Policies

Our continuous aggregates refresh automatically on a schedule:

| CAGG | Refresh Interval | Start Offset | End Offset | Max Latency |
|------|------------------|--------------|------------|-------------|
| `leaderboard_30min` | 30 minutes | 1 hour | 10 minutes | 40 minutes |
| `volume_hourly` | 1 hour | 2 hours | 10 minutes | 70 minutes |
| `pnl_daily` | 6 hours | 1 day | 1 hour | 7 hours |
| `trader_metrics_daily` | 24 hours | 2 days | 1 hour | 25 hours |

**Understanding Offsets:**
- `start_offset`: How far back to refresh (catches late data)
- `end_offset`: Don't refresh too-recent data (still being written)
- `schedule_interval`: How often the refresh runs

```sql
-- Example refresh policy configuration
SELECT add_continuous_aggregate_policy('leaderboard_30min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '30 minutes'
);
```

### Real-Time vs Near-Real-Time

| Data Type | Latency | Method |
|-----------|---------|--------|
| Current prices | <1 second | Direct table query |
| Open positions | <5 seconds | Direct table query |
| Leaderboard rankings | 30 minutes | Continuous aggregate |
| Win rate / metrics | 30 minutes | Continuous aggregate |
| Daily summaries | 6 hours | Continuous aggregate |

---

## Part 10: Privacy & Security

### Wallet Address Handling

**What We Collect:**
- Wallet addresses (public blockchain data)
- Trade history (public blockchain data)
- Position information (public blockchain data)

**What We DON'T Collect:**
- Personal identity information
- Email addresses
- IP addresses
- Private keys (obviously)

### Pseudonymous Analytics

All analytics are based on wallet addresses, which are pseudonymous by design:
- `0x742d35Cc6634C0532925a3b844Bc9e7595f8bB98` is just a string
- We don't attempt to link wallets to real identities
- You control your privacy by using multiple wallets

### Data Access Controls

```sql
-- Row-level security for multi-tenant access
CREATE POLICY wallet_isolation ON hl_user_fills
    FOR SELECT
    USING (address = current_setting('app.current_wallet'));

-- API keys have scoped permissions
GRANT SELECT ON leaderboard TO api_readonly;
REVOKE ALL ON wallet_data FROM api_readonly;
```

### Security Measures

| Layer | Protection |
|-------|------------|
| Network | TLS 1.3 encryption in transit |
| Database | Encrypted at rest (AES-256) |
| API | Rate limiting, authentication required |
| Infrastructure | VPC isolation, security groups |
| Access | Role-based permissions, audit logging |

---

## Part 11: Compression & Storage Optimization

### Why Compression Matters

Our self-managed TimescaleDB uses native compression to reduce storage by up to **90%** while maintaining query performance.

### Compression Statistics

| Table | Uncompressed Size | Compressed Size | Ratio |
|-------|------------------|-----------------|-------|
| hl_user_fills | ~800 GB | ~150 GB | 5.3x |
| hl_orderbook | ~600 GB | ~100 GB | 6x |
| perp_pnl_histories | ~200 GB | ~40 GB | 5x |
| leaderboard | ~400 GB | ~80 GB | 5x |

### Compression Configuration

```sql
-- Example compression settings for hl_user_fills
ALTER TABLE hl_user_fills SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'token',
    timescaledb.compress_orderby = 'address ASC, "timestampUTC" DESC'
);

-- Compression policy: compress chunks older than 7 days
SELECT add_compression_policy('hl_user_fills', INTERVAL '7 days');
```

---

## Part 12: Cost Savings Analysis

### Timescale Cloud vs Self-Managed

Our migration from Timescale Cloud to self-managed EC2 resulted in significant cost savings:

| Metric | Timescale Cloud | Self-Managed EC2 |
|--------|-----------------|------------------|
| **Monthly Cost** | ~$3,000/month | ~$800/month |
| **Annual Savings** | - | **$26,400/year** |
| **Storage** | 2.5 TB | 2.5 TB |
| **Performance** | Excellent | Excellent |
| **Control** | Limited | Full |

### Infrastructure Specifications

**Production Server:**
- Instance: r6i.2xlarge (8 vCPU, 64 GB RAM)
- Storage: 3 TB gp3 SSD (12,000 IOPS)
- PostgreSQL 16 + TimescaleDB 2.x
- Region: ap-northeast-1 (Tokyo)

---

## Part 13: API & Integration

### REST API Examples

**Get Top Traders by PnL (Last 30 Days):**
```sql
SELECT
    address,
    SUM(closedPnl) as total_pnl,
    COUNT(*) as trade_count,
    COUNT(*) FILTER (WHERE closedPnl > 0) as winning_trades,
    ROUND(100.0 * COUNT(*) FILTER (WHERE closedPnl > 0) / COUNT(*), 2) as win_rate
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    AND closedPnl IS NOT NULL
GROUP BY address
HAVING COUNT(*) >= 10
ORDER BY total_pnl DESC
LIMIT 100;
```

**Get Market Volume Trends:**
```sql
SELECT
    time_bucket('1 hour', "timestampUTC") as hour,
    token,
    SUM(size * price) as volume,
    COUNT(*) as trade_count
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
GROUP BY hour, token
ORDER BY hour DESC, volume DESC;
```

**Get Liquidation Risk Zones:**
```sql
SELECT
    token,
    ROUND(liquidation_price, -2) as price_zone,  -- Round to nearest 100
    COUNT(*) as liquidation_count,
    SUM(size) as total_size_liquidated
FROM hl_liquidation_orders
WHERE timestamp >= NOW() - INTERVAL '30 days'
GROUP BY token, ROUND(liquidation_price, -2)
ORDER BY token, price_zone;
```

### GraphQL Schema Example

```graphql
type Trader {
  address: String!
  totalPnl: Float!
  winRate: Float!
  tradeCount: Int!
  tier: String!
  badge: String!
  sharpeRatio: Float
  maxDrawdown: Float
}

type MarketVolume {
  token: String!
  hour: DateTime!
  volume: Float!
  tradeCount: Int!
}

type Leaderboard {
  traders: [Trader!]!
  lastUpdated: DateTime!
}

type Query {
  # Get leaderboard with optional filters
  leaderboard(
    timeframe: Timeframe = MONTHLY
    limit: Int = 100
    minTrades: Int = 10
  ): Leaderboard!

  # Get specific trader analytics
  trader(address: String!): Trader

  # Get market volume trends
  marketVolume(
    token: String!
    interval: Interval = HOURLY
    days: Int = 7
  ): [MarketVolume!]!
}

enum Timeframe {
  DAILY
  WEEKLY
  MONTHLY
  ALL_TIME
}

enum Interval {
  MINUTE
  HOURLY
  DAILY
}
```

### JavaScript/TypeScript Integration

```typescript
// Example: Fetching leaderboard data with caching
import { Pool } from 'pg';

const pool = new Pool({
  host: 'your-timescaledb-host',
  port: 5432,
  database: 'tsdb',
  user: 'api_readonly',
  password: process.env.DB_PASSWORD,
  ssl: { rejectUnauthorized: false }
});

interface TraderMetrics {
  address: string;
  totalPnl: number;
  winRate: number;
  tradeCount: number;
}

async function getTopTraders(days: number = 30, limit: number = 100): Promise<TraderMetrics[]> {
  const query = `
    SELECT
      address,
      SUM(closedPnl) as total_pnl,
      ROUND(100.0 * COUNT(*) FILTER (WHERE closedPnl > 0) / COUNT(*), 2) as win_rate,
      COUNT(*) as trade_count
    FROM leaderboard_30min  -- Use CAGG for performance
    WHERE bucket >= NOW() - INTERVAL '${days} days'
    GROUP BY address
    HAVING COUNT(*) >= 10
    ORDER BY total_pnl DESC
    LIMIT $1
  `;

  const result = await pool.query(query, [limit]);
  return result.rows.map(row => ({
    address: row.address,
    totalPnl: parseFloat(row.total_pnl),
    winRate: parseFloat(row.win_rate),
    tradeCount: parseInt(row.trade_count)
  }));
}

// Example: Real-time position tracking
async function getOpenPositions(address: string) {
  const query = `
    SELECT
      token,
      size,
      entry_price,
      leverage,
      unrealized_pnl
    FROM perp_asset_positions
    WHERE address = $1 AND size != 0
  `;

  return await pool.query(query, [address]);
}
```

### Python Integration

```python
import psycopg2
import pandas as pd
from datetime import datetime, timedelta

class HyperliquidAnalytics:
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)

    def get_trader_performance(self, address: str, days: int = 30) -> dict:
        """Get comprehensive trader metrics"""
        query = """
            WITH trades AS (
                SELECT
                    closedPnl,
                    size * price as trade_value
                FROM hl_user_fills
                WHERE address = %s
                  AND "timestampUTC" >= NOW() - INTERVAL '%s days'
                  AND closedPnl IS NOT NULL
            )
            SELECT
                COUNT(*) as total_trades,
                COUNT(*) FILTER (WHERE closedPnl > 0) as winning_trades,
                SUM(closedPnl) as total_pnl,
                AVG(closedPnl) as avg_pnl,
                STDDEV(closedPnl) as pnl_stddev,
                MAX(closedPnl) as best_trade,
                MIN(closedPnl) as worst_trade
            FROM trades
        """

        df = pd.read_sql(query, self.conn, params=[address, days])

        if df.empty or df['total_trades'].iloc[0] == 0:
            return None

        row = df.iloc[0]
        return {
            'address': address,
            'total_trades': int(row['total_trades']),
            'win_rate': row['winning_trades'] / row['total_trades'] * 100,
            'total_pnl': float(row['total_pnl']),
            'avg_pnl': float(row['avg_pnl']),
            'sharpe_ratio': float(row['avg_pnl'] / row['pnl_stddev']) if row['pnl_stddev'] > 0 else None,
            'best_trade': float(row['best_trade']),
            'worst_trade': float(row['worst_trade'])
        }

    def get_volume_by_market(self, hours: int = 24) -> pd.DataFrame:
        """Get trading volume breakdown by market"""
        query = """
            SELECT
                token,
                SUM(size * price) as volume,
                COUNT(*) as trade_count,
                COUNT(DISTINCT address) as unique_traders
            FROM hl_user_fills
            WHERE "timestampUTC" >= NOW() - INTERVAL '%s hours'
            GROUP BY token
            ORDER BY volume DESC
        """
        return pd.read_sql(query, self.conn, params=[hours])

# Usage example
analytics = HyperliquidAnalytics("postgresql://user:pass@host:5432/tsdb")
trader_stats = analytics.get_trader_performance("0x742d35Cc...")
print(f"Win Rate: {trader_stats['win_rate']:.1f}%")
print(f"Sharpe Ratio: {trader_stats['sharpe_ratio']:.2f}")
```

---

## Part 14: Using Analytics to Improve Your Trading

### The Feedback Loop

**Step 1: Weekly Review Checklist**
- [ ] Check win rate trend (improving or declining?)
- [ ] Review mean PnL per trade (positive or negative?)
- [ ] Identify best markets (where are you most profitable?)
- [ ] Analyze worst markets (where are you losing money?)
- [ ] Study biggest wins (what did you do right?)
- [ ] Examine biggest losses (what went wrong?)

**Step 2: Identify Patterns**
Run this query to find your personal patterns:

```sql
-- Find your best and worst trading conditions
SELECT
    token,
    side,
    EXTRACT(DOW FROM "timestampUTC") as day_of_week,
    EXTRACT(HOUR FROM "timestampUTC") as hour_of_day,
    COUNT(*) as trade_count,
    SUM(closedPnl) as total_pnl,
    AVG(closedPnl) as avg_pnl,
    ROUND(100.0 * COUNT(*) FILTER (WHERE closedPnl > 0) / COUNT(*), 2) as win_rate
FROM hl_user_fills
WHERE address = '0x...'
    AND closedPnl IS NOT NULL
GROUP BY token, side,
         EXTRACT(DOW FROM "timestampUTC"),
         EXTRACT(HOUR FROM "timestampUTC")
HAVING COUNT(*) >= 5
ORDER BY avg_pnl DESC;
```

### Common Insights and Actions

| Insight | Analysis | Action |
|---------|----------|--------|
| "I have 65% win rate but I'm losing money" | Losses bigger than wins | Implement tighter stop losses, let winners run |
| "I make money Mon-Thu but lose Fridays" | Weekend positioning issues | Close positions Thursday, take weekends off |
| "I'm profitable until I trade altcoins" | Altcoin volatility hurting you | Stick to BTC/ETH or reduce altcoin position sizes |
| "Position size >20% of account = losses" | Emotional attachment clouds judgment | Max 10% position size rule, no exceptions |
| "Sharpe Ratio is 0.3 despite being profitable" | Very volatile, risky path | Reduce position sizes and leverage |

---

## Conclusion: Data-Driven Trading Success

### Why This Matters

In the end, trading is a game of probabilities, risk management, and continuous improvement. You can't improve what you don't measure.

Our analytics platform gives you:

- **Complete transparency** - Every number, every trade, every metric
- **Competitive context** - Where you rank vs hundreds of thousands of traders
- **Pattern recognition** - What's working, what's not
- **Progress tracking** - Are you improving or declining?
- **Milestone celebration** - Recognition for achievements
- **Honest feedback** - Real vs paper profits, no BS

### The Path Forward

**For Beginners (Minnow/Barracuda Tiers):**
- Focus on learning, not earning
- Study your metrics to understand your mistakes
- Use small position sizes while building skill
- Celebrate small wins (first profitable week!)

**For Intermediate Traders (Dolphin/Shark Tiers):**
- Refine your edge using analytics
- Identify your best markets and setups
- Scale position sizes as consistency improves
- Target the next tier milestone

**For Advanced Traders (Whale Tiers):**
- Optimize risk-adjusted returns (Sharpe Ratio)
- Diversify strategies and markets
- Mentor others using your data
- Compete for top leaderboard positions

**For Elite Traders (Blue Whale Tiers):**
- Maintain your position
- Share insights with the community
- Use your platform influence responsibly

---

### Technical Appendix: Database Schema Summary

| Object Type | Count | Examples |
|-------------|-------|----------|
| **Hypertables** | 57 | hl_user_fills, perp_pnl_histories, leaderboard |
| **Continuous Aggregates** | 131 | leaderboard_30min, volume_hourly, pnl_daily |
| **Materialized Views** | 16 | sharpe_and_sortino_ratio, leaderboard_complete |
| **Regular Tables** | 130 | wallet_data, traders, positions |
| **Total Database Size** | 2.5 TB | - |
| **Compression Ratio** | ~5:1 | ~500 GB actual storage |

---

*Analytics updated every 30 minutes. Rankings refresh in real-time. Your trading journey is being recorded. Make it count.*

*Powered by PostgreSQL 16 + TimescaleDB on self-managed infrastructure.*
