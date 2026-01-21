# Advanced Trading Analytics with Hyperliquid Data: A Quantitative Analysis Playbook

**The Data-Driven Edge for Serious Traders**

This playbook provides production-ready SQL queries and analytics frameworks for extracting alpha from Hyperliquid trading data. Each section includes real queries you can run against TimescaleDB to gain quantitative insights.

---

## Table of Contents

1. [Backtesting Framework](#part-1-backtesting-framework)
2. [Whale Watching Analytics](#part-2-whale-watching-analytics)
3. [Correlation & Copy-Trading Detection](#part-3-correlation--copy-trading-detection)
4. [Market Microstructure Analysis](#part-4-market-microstructure-analysis)
5. [Portfolio Risk Metrics (VaR, CVaR, MDD)](#part-5-portfolio-risk-metrics)
6. [Technical Indicators (RSI, Moving Averages)](#part-6-technical-indicators)
7. [Time-of-Day Analysis](#part-7-time-of-day-analysis)
8. [Liquidation Cascade Analysis](#part-8-liquidation-cascade-analysis)
9. [Funding Rate Arbitrage](#part-9-funding-rate-arbitrage)
10. [Trader Segmentation & Clustering](#part-10-trader-segmentation--clustering)
11. [Orderbook Depth Analysis](#part-11-orderbook-depth-analysis)
12. [Open Interest Analytics](#part-12-open-interest-analytics)
13. [Capital Flow Analysis](#part-13-capital-flow-analysis)
14. [Using Pre-built CAGGs](#part-14-using-pre-built-caggs)

---

## Part 1: Backtesting Framework

### The Theory

Backtesting is the cornerstone of systematic trading - it allows you to test a hypothesis against historical data before risking real capital. However, backtesting is fraught with pitfalls that can lead to overfitting and false confidence.

#### Why Backtesting Matters

In quantitative finance, backtesting serves three critical purposes:
1. **Strategy Validation**: Does the strategy have statistical edge, or is it noise?
2. **Risk Assessment**: What are the worst-case scenarios (drawdowns, losing streaks)?
3. **Parameter Optimization**: What settings maximize risk-adjusted returns?

#### Common Backtesting Biases to Avoid

| Bias | Description | Mitigation |
|------|-------------|------------|
| **Look-ahead Bias** | Using information that wouldn't be available at decision time | Use only data available at signal generation time |
| **Survivorship Bias** | Only testing on assets that exist today | Include delisted/defunct tokens in dataset |
| **Data Snooping** | Fitting strategy to specific historical patterns | Use out-of-sample testing, walk-forward analysis |
| **Transaction Cost Neglect** | Ignoring fees, slippage, spread | Model realistic execution costs (0.05-0.1% for crypto) |

#### Key Formulas

**Strategy Returns:**
```
Strategy Return = Σ(Position Size × Price Change) - Transaction Costs
```

**Kelly Criterion** (Optimal Position Sizing):
```
f* = (bp - q) / b

Where:
- f* = Fraction of capital to bet
- b = Win/loss ratio (average win / average loss)
- p = Win rate (probability of winning)
- q = 1 - p (probability of losing)
```

The Kelly formula maximizes the geometric growth rate of capital. However, full Kelly is aggressive - most traders use "half Kelly" (f*/2) or "quarter Kelly" for a smoother equity curve.

**Sharpe Ratio** (Risk-Adjusted Returns):
```
Sharpe = (Mean Return - Risk-Free Rate) / Standard Deviation of Returns
```

A Sharpe ratio above 1.0 is considered acceptable, above 2.0 is excellent. In crypto, with no clear risk-free rate, traders often use 0% or the stablecoin yield rate.

### SQL: Simulate a Simple Momentum Strategy

This query backtests a strategy that goes long when the 1-hour return exceeds 2%:

```sql
-- Backtest: Go long after 2% hourly gain, exit after 1 hour
WITH hourly_returns AS (
    SELECT
        time_bucket('1 hour', "timestampUTC") AS hour,
        token,
        first(price, "timestampUTC") AS open_price,
        last(price, "timestampUTC") AS close_price,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS hourly_return_pct
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY hour, token
),
signals AS (
    SELECT
        hour,
        token,
        hourly_return_pct,
        LEAD(hourly_return_pct) OVER (PARTITION BY token ORDER BY hour) AS next_hour_return,
        CASE WHEN hourly_return_pct > 2 THEN 'LONG'
             WHEN hourly_return_pct < -2 THEN 'SHORT'
             ELSE 'FLAT' END AS signal
    FROM hourly_returns
)
SELECT
    token,
    COUNT(*) FILTER (WHERE signal = 'LONG') AS long_signals,
    COUNT(*) FILTER (WHERE signal = 'SHORT') AS short_signals,

    -- Long performance
    AVG(next_hour_return) FILTER (WHERE signal = 'LONG') AS avg_long_return,
    STDDEV(next_hour_return) FILTER (WHERE signal = 'LONG') AS long_volatility,

    -- Short performance
    AVG(-next_hour_return) FILTER (WHERE signal = 'SHORT') AS avg_short_return,

    -- Win rate
    COUNT(*) FILTER (WHERE signal = 'LONG' AND next_hour_return > 0)::float /
        NULLIF(COUNT(*) FILTER (WHERE signal = 'LONG'), 0) * 100 AS long_win_rate_pct
FROM signals
WHERE signal != 'FLAT'
GROUP BY token
HAVING COUNT(*) FILTER (WHERE signal = 'LONG') >= 10
ORDER BY avg_long_return DESC
LIMIT 20;
```

### SQL: Calculate Hypothetical P&L with Position Sizing

```sql
-- Backtest with Kelly-optimal position sizing
WITH trade_results AS (
    SELECT
        address,
        token,
        "timestampUTC",
        side,
        size_usd,
        price,
        -- Calculate P&L per trade (simplified)
        CASE
            WHEN side = 'BUY' THEN
                (LEAD(price) OVER (PARTITION BY address, token ORDER BY "timestampUTC") - price) * size_usd / price
            ELSE
                (price - LEAD(price) OVER (PARTITION BY address, token ORDER BY "timestampUTC")) * size_usd / price
        END AS trade_pnl
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
),
trader_stats AS (
    SELECT
        address,
        COUNT(*) AS total_trades,
        SUM(CASE WHEN trade_pnl > 0 THEN 1 ELSE 0 END)::float / COUNT(*) AS win_rate,
        AVG(trade_pnl) FILTER (WHERE trade_pnl > 0) AS avg_win,
        AVG(ABS(trade_pnl)) FILTER (WHERE trade_pnl < 0) AS avg_loss,
        SUM(trade_pnl) AS total_pnl
    FROM trade_results
    WHERE trade_pnl IS NOT NULL
    GROUP BY address
    HAVING COUNT(*) >= 50
)
SELECT
    address,
    total_trades,
    ROUND(win_rate * 100, 2) AS win_rate_pct,
    ROUND(avg_win::numeric, 2) AS avg_win_usd,
    ROUND(avg_loss::numeric, 2) AS avg_loss_usd,
    ROUND(total_pnl::numeric, 2) AS total_pnl_usd,

    -- Kelly Criterion: f* = (bp - q) / b where b = avg_win/avg_loss, p = win_rate, q = 1-p
    ROUND(
        ((avg_win / NULLIF(avg_loss, 0)) * win_rate - (1 - win_rate)) /
        NULLIF(avg_win / NULLIF(avg_loss, 0), 0) * 100, 2
    ) AS kelly_fraction_pct
FROM trader_stats
WHERE total_pnl > 1000
ORDER BY total_pnl DESC
LIMIT 50;
```

---

## Part 2: Whale Watching Analytics

### The Theory

In financial markets, large traders (commonly called "whales") have disproportionate influence on price movements. Understanding whale behavior is central to market microstructure theory and can provide significant trading edge.

#### Why Whales Matter

Whales move markets through several mechanisms:

1. **Information Asymmetry**: Large traders often have superior information due to:
   - Better research and analysis resources
   - Access to institutional-grade data feeds
   - Relationships with market makers and protocol teams
   - More sophisticated trading infrastructure

2. **Price Impact**: Kyle's Lambda model (1985) quantifies how large orders move prices:
   ```
   ΔP = λ × OrderSize

   Where λ (lambda) = Market impact coefficient
   ```
   In thin crypto markets, λ can be substantial - a $1M order might move price 1-5%.

3. **Liquidity Provision**: Whales often act as de facto market makers, providing liquidity during volatile periods.

#### Whale Classification System

| Tier | 30-Day Volume | Characteristics |
|------|---------------|-----------------|
| **Mega Whale** | >$100M | Institutional, multi-strategy, likely market maker |
| **Whale** | $10M-$100M | Sophisticated trader, systematic strategies |
| **Dolphin** | $1M-$10M | Active trader, likely profitable |
| **Fish** | <$1M | Retail, noise trading |

#### Key Metrics to Track

**Net Flow (Delta)**: Measures directional pressure
```
Net Flow = Σ(Buy Volume) - Σ(Sell Volume)
```

**Flow-Price Correlation**: Tests if whale activity predicts price
```
Correlation(Net Flow_t, Return_t+1)
```

A positive correlation suggests whale activity has predictive power.

**Whale Concentration**: How much volume comes from top N addresses
```
Concentration Ratio = Volume_top_N / Total_Volume
```

High concentration suggests the market is dominated by a few players.

### SQL: Identify Whales by Trading Volume

```sql
-- Top whales by 30-day trading volume
WITH whale_candidates AS (
    SELECT
        address,
        SUM(size_usd) AS total_volume_usd,
        COUNT(*) AS trade_count,
        COUNT(DISTINCT token) AS tokens_traded,
        AVG(size_usd) AS avg_trade_size,
        MAX(size_usd) AS largest_trade,

        -- Trading style metrics
        SUM(size_usd) FILTER (WHERE side = 'BUY') AS buy_volume,
        SUM(size_usd) FILTER (WHERE side = 'SELL') AS sell_volume
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY address
)
SELECT
    address,
    ROUND(total_volume_usd / 1e6, 2) AS volume_millions,
    trade_count,
    tokens_traded,
    ROUND(avg_trade_size::numeric, 2) AS avg_trade_usd,
    ROUND(largest_trade::numeric, 2) AS max_trade_usd,
    ROUND((buy_volume - sell_volume) / NULLIF(total_volume_usd, 0) * 100, 2) AS net_bias_pct,

    -- Whale classification
    CASE
        WHEN total_volume_usd > 100000000 THEN 'MEGA_WHALE'
        WHEN total_volume_usd > 10000000 THEN 'WHALE'
        WHEN total_volume_usd > 1000000 THEN 'DOLPHIN'
        ELSE 'FISH'
    END AS trader_tier
FROM whale_candidates
WHERE total_volume_usd > 1000000
ORDER BY total_volume_usd DESC
LIMIT 100;
```

### SQL: Track Whale Entry/Exit Patterns

```sql
-- Whale position changes with price impact analysis
WITH whale_trades AS (
    SELECT
        f.address,
        f.token,
        time_bucket('1 hour', f."timestampUTC") AS trade_hour,
        SUM(CASE WHEN f.side = 'BUY' THEN f.size_usd ELSE -f.size_usd END) AS net_position_change,
        SUM(f.size_usd) AS total_volume
    FROM hl_user_fills f
    WHERE f.address IN (
        -- Only top 100 whales
        SELECT address FROM (
            SELECT address, SUM(size_usd) as vol
            FROM hl_user_fills
            WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
            GROUP BY address
            ORDER BY vol DESC
            LIMIT 100
        ) top_whales
    )
    AND f."timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY f.address, f.token, trade_hour
    HAVING SUM(f.size_usd) > 50000  -- Only significant moves
),
price_moves AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        first(price, "timestampUTC") AS open_price,
        last(price, "timestampUTC") AS close_price,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS hour_return_pct
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY token, hour
)
SELECT
    w.token,
    SUM(w.net_position_change) AS total_whale_flow,
    COUNT(DISTINCT w.address) AS whale_count,
    AVG(p.hour_return_pct) AS avg_price_move_pct,

    -- Correlation between whale flow and price
    CORR(w.net_position_change, p.hour_return_pct) AS flow_price_correlation,

    -- Next hour return after whale activity
    AVG(LEAD(p.hour_return_pct) OVER (PARTITION BY w.token ORDER BY w.trade_hour)) AS avg_next_hour_return
FROM whale_trades w
JOIN price_moves p ON w.token = p.token AND w.trade_hour = p.hour
GROUP BY w.token
HAVING COUNT(*) >= 10
ORDER BY ABS(total_whale_flow) DESC
LIMIT 30;
```

### SQL: Whale Alert System - Large Position Changes

```sql
-- Real-time whale alerts: Detect large position changes
SELECT
    address,
    token,
    "timestampUTC",
    side,
    size_usd,
    price,

    -- Rolling 1-hour volume for this address
    SUM(size_usd) OVER (
        PARTITION BY address, token
        ORDER BY "timestampUTC"
        RANGE BETWEEN INTERVAL '1 hour' PRECEDING AND CURRENT ROW
    ) AS rolling_1h_volume,

    -- Is this an unusual trade?
    CASE WHEN size_usd > 100000 THEN 'LARGE_TRADE'
         WHEN size_usd > 50000 THEN 'MEDIUM_TRADE'
         ELSE 'NORMAL' END AS trade_significance
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
  AND size_usd > 50000
ORDER BY size_usd DESC
LIMIT 50;
```

---

## Part 3: Correlation & Copy-Trading Detection

### The Theory

Copy-trading detection identifies traders whose actions are highly correlated - either through automated copy systems or coordinated trading. This is a form of **network analysis** combined with **time-series correlation** that can reveal hidden relationships in trading data.

#### Why Correlation Analysis Matters

1. **Alpha Identification**: Finding profitable traders to follow
2. **Risk Management**: Detecting coordinated manipulation
3. **Strategy Discovery**: Understanding market dynamics
4. **Compliance**: Identifying potential market manipulation

#### Correlation Measures

**Pearson Correlation** (linear relationships):
```
ρ(X,Y) = Cov(X,Y) / (σX × σY)
```
- Range: -1 to +1
- +1: Perfect positive correlation (same direction)
- -1: Perfect negative correlation (opposite direction)
- 0: No linear relationship

**Spearman Correlation** (rank-based, more robust to outliers):
```
ρs = 1 - (6 × Σd²) / (n × (n² - 1))
```
Where d = difference between ranks of corresponding values

#### Lead-Lag Relationships

In copy-trading, one trader (the "leader") acts first, and another (the "follower") copies after a delay. This is modeled as:
```
Signal_follower(t) = α + β × Signal_leader(t - lag) + ε
```

**Cross-Correlation** at lag k:
```
CCF(k) = Correlation(X_t, Y_{t+k})
```

| Correlation Threshold | Interpretation |
|-----------------------|----------------|
| > 0.9 | Very likely copy-trading or same entity |
| 0.7 - 0.9 | Possible copy-trading or similar strategy |
| 0.5 - 0.7 | Correlated strategies, may follow same signals |
| < -0.7 | Inverse trading (hedging or adversarial) |

#### Statistical Significance

Not all correlations are meaningful. Use t-test for significance:
```
t = r × √((n-2) / (1-r²))
```
With n-2 degrees of freedom. A p-value < 0.05 indicates statistical significance.

#### Key Detection Methods

- **Temporal correlation**: Do traders enter/exit at similar times?
- **Position correlation**: Do they hold similar positions?
- **Lag analysis**: Does one trader consistently follow another?
- **Granger Causality**: Does trader A's activity help predict trader B's?

### SQL: Detect Potential Copy-Trading Pairs

```sql
-- Find trader pairs with high trade correlation
WITH trader_activity AS (
    SELECT
        address,
        token,
        time_bucket('5 minutes', "timestampUTC") AS time_bucket,
        SUM(CASE WHEN side = 'BUY' THEN size_usd ELSE -size_usd END) AS net_flow
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY address, token, time_bucket
    HAVING ABS(SUM(CASE WHEN side = 'BUY' THEN size_usd ELSE -size_usd END)) > 1000
),
trader_pairs AS (
    SELECT
        a1.address AS trader_a,
        a2.address AS trader_b,
        a1.token,
        COUNT(*) AS shared_buckets,
        CORR(a1.net_flow, a2.net_flow) AS flow_correlation,
        AVG(ABS(a1.net_flow - a2.net_flow)) AS avg_flow_difference
    FROM trader_activity a1
    JOIN trader_activity a2
        ON a1.token = a2.token
        AND a1.time_bucket = a2.time_bucket
        AND a1.address < a2.address  -- Avoid duplicates
    GROUP BY a1.address, a2.address, a1.token
    HAVING COUNT(*) >= 20  -- At least 20 matching time buckets
)
SELECT
    trader_a,
    trader_b,
    token,
    shared_buckets,
    ROUND(flow_correlation::numeric, 4) AS correlation,
    ROUND(avg_flow_difference::numeric, 2) AS avg_diff_usd,

    CASE
        WHEN flow_correlation > 0.9 THEN 'LIKELY_COPY'
        WHEN flow_correlation > 0.7 THEN 'POSSIBLE_COPY'
        WHEN flow_correlation < -0.7 THEN 'INVERSE_TRADING'
        ELSE 'INDEPENDENT'
    END AS relationship_type
FROM trader_pairs
WHERE ABS(flow_correlation) > 0.7
ORDER BY flow_correlation DESC
LIMIT 50;
```

### SQL: Lag Analysis - Who Leads, Who Follows

```sql
-- Identify leader-follower relationships
WITH trader_signals AS (
    SELECT
        address,
        token,
        time_bucket('1 minute', "timestampUTC") AS minute,
        SUM(CASE WHEN side = 'BUY' THEN 1 ELSE -1 END) AS signal
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
    GROUP BY address, token, minute
),
lagged_correlation AS (
    SELECT
        t1.address AS potential_leader,
        t2.address AS potential_follower,
        t1.token,

        -- Correlation with different lag offsets
        CORR(t1.signal, t2.signal) AS same_time_corr,
        CORR(t1.signal, LAG(t2.signal, 1) OVER (PARTITION BY t2.address, t2.token ORDER BY t2.minute)) AS lag_1m_corr,
        CORR(t1.signal, LAG(t2.signal, 5) OVER (PARTITION BY t2.address, t2.token ORDER BY t2.minute)) AS lag_5m_corr,
        CORR(t1.signal, LAG(t2.signal, 10) OVER (PARTITION BY t2.address, t2.token ORDER BY t2.minute)) AS lag_10m_corr
    FROM trader_signals t1
    JOIN trader_signals t2
        ON t1.token = t2.token
        AND t1.minute = t2.minute
        AND t1.address != t2.address
    GROUP BY t1.address, t2.address, t1.token
    HAVING COUNT(*) >= 30
)
SELECT
    potential_leader,
    potential_follower,
    token,
    ROUND(same_time_corr::numeric, 3) AS corr_0m,
    ROUND(lag_1m_corr::numeric, 3) AS corr_1m_lag,
    ROUND(lag_5m_corr::numeric, 3) AS corr_5m_lag,
    ROUND(lag_10m_corr::numeric, 3) AS corr_10m_lag,

    -- Identify if follower lags leader
    CASE
        WHEN COALESCE(lag_5m_corr, 0) > same_time_corr + 0.1 THEN 'FOLLOWER_LAGS_5M'
        WHEN COALESCE(lag_1m_corr, 0) > same_time_corr + 0.1 THEN 'FOLLOWER_LAGS_1M'
        ELSE 'SIMULTANEOUS'
    END AS relationship
FROM lagged_correlation
WHERE ABS(COALESCE(lag_5m_corr, 0)) > 0.5 OR ABS(same_time_corr) > 0.5
ORDER BY GREATEST(ABS(same_time_corr), ABS(COALESCE(lag_5m_corr, 0))) DESC
LIMIT 50;
```

---

## Part 4: Market Microstructure Analysis

### The Theory

Market microstructure is the study of how trading mechanisms and protocols affect price formation, transaction costs, and market efficiency. This field, pioneered by researchers like Kyle (1985), Glosten & Milgrom (1985), and O'Hara (1995), is essential for understanding trading costs and market quality.

#### Core Concepts

**1. Bid-Ask Spread**
The spread represents the cost of immediacy - the premium paid to trade immediately rather than wait.

```
Spread = Ask Price - Bid Price
Spread (bps) = (Ask - Bid) / Mid × 10000
```

Components of the spread:
- **Adverse Selection**: Compensation for trading against informed traders
- **Inventory Risk**: Cost of holding unwanted positions
- **Order Processing**: Fixed costs of market making

**2. Effective Spread vs Quoted Spread**
```
Quoted Spread = Best Ask - Best Bid
Effective Spread = 2 × |Trade Price - Mid Price|
```
The effective spread captures the actual transaction cost, which may differ from the quoted spread due to price improvement or execution outside NBBO.

**3. Order Flow Imbalance (OFI)**
Order flow imbalance measures the pressure differential between buyers and sellers:

```
OFI = (Buy Volume - Sell Volume) / Total Volume
```

Range: -1 (all selling) to +1 (all buying)

**Kyle's Lambda Model** predicts that price changes are proportional to order flow:
```
ΔP = λ × (Signed Order Flow) + ε
```
Where λ captures market impact per unit of order flow.

**4. Trade Clustering & HFT Detection**

High-frequency traders create distinctive patterns:
- **Burst Trading**: Many trades in short intervals (<100ms)
- **Quote Stuffing**: Rapid order placement/cancellation
- **Latency Arbitrage**: Exploiting speed advantages

The **coefficient of variation** in inter-trade times indicates clustering:
```
CV = Standard Deviation / Mean
```
CV > 1 suggests clustered/bursty trading patterns.

#### Key Microstructure Metrics

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| Spread (bps) | (Ask-Bid)/Mid × 10000 | Liquidity cost |
| Amihud Illiquidity | |Return| / Volume | Price impact per $ |
| Roll Spread | 2 × √(-Cov(Δp_t, Δp_{t-1})) | Implied spread |
| Kyle's Lambda | Regress ΔP on Order Flow | Market impact |

#### Key Analysis Areas

- **Spread analysis**: The cost of immediacy
- **Order flow imbalance**: Buy vs sell pressure
- **Trade clustering**: High-frequency patterns
- **Price impact**: How large orders move prices

### SQL: Bid-Ask Spread Analysis

```sql
-- Analyze effective spreads from trade data
WITH trade_pairs AS (
    SELECT
        token,
        time_bucket('1 minute', "timestampUTC") AS minute,
        MIN(price) AS low_price,
        MAX(price) AS high_price,
        first(price, "timestampUTC") AS open_price,
        last(price, "timestampUTC") AS close_price,
        COUNT(*) AS trade_count,
        SUM(size_usd) AS volume_usd
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
    GROUP BY token, minute
)
SELECT
    token,
    COUNT(*) AS total_minutes,
    ROUND(AVG((high_price - low_price) / NULLIF(open_price, 0) * 10000)::numeric, 2) AS avg_spread_bps,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY (high_price - low_price) / NULLIF(open_price, 0) * 10000)::numeric, 2) AS median_spread_bps,
    ROUND(AVG(trade_count)::numeric, 1) AS avg_trades_per_min,
    ROUND(AVG(volume_usd)::numeric, 2) AS avg_volume_per_min,

    -- Volatility-adjusted spread
    ROUND(AVG((high_price - low_price) / NULLIF(open_price, 0) * 10000) /
          NULLIF(STDDEV((close_price - open_price) / NULLIF(open_price, 0) * 100), 0)::numeric, 4) AS spread_vol_ratio
FROM trade_pairs
GROUP BY token
HAVING COUNT(*) >= 100
ORDER BY avg_spread_bps ASC
LIMIT 30;
```

### SQL: Order Flow Imbalance

```sql
-- Calculate order flow imbalance and its predictive power
WITH flow_data AS (
    SELECT
        token,
        time_bucket('5 minutes', "timestampUTC") AS bucket,
        SUM(size_usd) FILTER (WHERE side = 'BUY') AS buy_volume,
        SUM(size_usd) FILTER (WHERE side = 'SELL') AS sell_volume,
        SUM(size_usd) AS total_volume,
        last(price, "timestampUTC") AS close_price
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY token, bucket
),
imbalance_calc AS (
    SELECT
        token,
        bucket,
        (COALESCE(buy_volume, 0) - COALESCE(sell_volume, 0)) / NULLIF(total_volume, 0) AS order_flow_imbalance,
        (close_price - LAG(close_price) OVER (PARTITION BY token ORDER BY bucket)) /
            NULLIF(LAG(close_price) OVER (PARTITION BY token ORDER BY bucket), 0) * 100 AS return_pct,
        LEAD((close_price - LAG(close_price) OVER (PARTITION BY token ORDER BY bucket)) /
            NULLIF(LAG(close_price) OVER (PARTITION BY token ORDER BY bucket), 0) * 100)
            OVER (PARTITION BY token ORDER BY bucket) AS next_return_pct
    FROM flow_data
)
SELECT
    token,
    COUNT(*) AS observations,

    -- Imbalance statistics
    ROUND(AVG(order_flow_imbalance)::numeric, 4) AS avg_imbalance,
    ROUND(STDDEV(order_flow_imbalance)::numeric, 4) AS imbalance_std,

    -- Predictive power
    ROUND(CORR(order_flow_imbalance, return_pct)::numeric, 4) AS contemp_correlation,
    ROUND(CORR(order_flow_imbalance, next_return_pct)::numeric, 4) AS predictive_correlation,

    -- Returns by imbalance quintile
    ROUND(AVG(next_return_pct) FILTER (WHERE order_flow_imbalance > 0.3)::numeric, 4) AS return_high_buy_pressure,
    ROUND(AVG(next_return_pct) FILTER (WHERE order_flow_imbalance < -0.3)::numeric, 4) AS return_high_sell_pressure
FROM imbalance_calc
WHERE order_flow_imbalance IS NOT NULL AND next_return_pct IS NOT NULL
GROUP BY token
HAVING COUNT(*) >= 500
ORDER BY ABS(CORR(order_flow_imbalance, next_return_pct)) DESC
LIMIT 20;
```

### SQL: Trade Clustering Analysis

```sql
-- Detect trade clustering patterns (HFT activity indicator)
WITH inter_trade_times AS (
    SELECT
        token,
        "timestampUTC",
        EXTRACT(EPOCH FROM "timestampUTC" - LAG("timestampUTC") OVER (PARTITION BY token ORDER BY "timestampUTC")) * 1000 AS ms_since_last_trade,
        size_usd
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
)
SELECT
    token,
    COUNT(*) AS trade_count,

    -- Inter-trade time statistics
    ROUND(AVG(ms_since_last_trade)::numeric, 2) AS avg_ms_between_trades,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ms_since_last_trade)::numeric, 2) AS median_ms,
    ROUND(MIN(ms_since_last_trade)::numeric, 2) AS min_ms,

    -- Clustering metrics
    COUNT(*) FILTER (WHERE ms_since_last_trade < 100) AS trades_under_100ms,
    COUNT(*) FILTER (WHERE ms_since_last_trade < 1000) AS trades_under_1s,

    -- Coefficient of variation (higher = more clustered)
    ROUND((STDDEV(ms_since_last_trade) / NULLIF(AVG(ms_since_last_trade), 0))::numeric, 3) AS clustering_coefficient,

    -- Volume in clusters
    SUM(size_usd) FILTER (WHERE ms_since_last_trade < 100) AS cluster_volume_usd
FROM inter_trade_times
WHERE ms_since_last_trade IS NOT NULL
GROUP BY token
ORDER BY trade_count DESC
LIMIT 20;
```

---

## Part 5: Portfolio Risk Metrics

### The Theory

Risk management is the cornerstone of professional trading. The legendary trader Ed Seykota said: "There are old traders and bold traders, but there are very few old, bold traders." Understanding and managing risk is what separates sustainable traders from those who eventually blow up.

#### Why Risk Metrics Matter

1. **Capital Preservation**: The first rule of trading is don't lose money; the second rule is don't forget rule one
2. **Position Sizing**: Risk metrics inform how much to bet on each trade
3. **Strategy Evaluation**: Compare strategies on a risk-adjusted basis, not just returns
4. **Regulatory Compliance**: Institutional traders must report risk metrics to regulators

#### Value at Risk (VaR)

VaR answers the question: "What is the maximum I can lose with X% confidence over time period T?"

**Three Methods to Calculate VaR:**

1. **Parametric VaR** (Variance-Covariance Method):
```
VaR(α) = μ - z(α) × σ

Where:
- μ = Expected return (often assumed 0 for short periods)
- z(α) = Z-score for confidence level (1.645 for 95%, 2.326 for 99%)
- σ = Standard deviation of returns
```

2. **Historical VaR** (Non-parametric):
```
VaR(95%) = 5th percentile of historical return distribution
```
Simply sort returns and take the cutoff at the desired percentile.

3. **Monte Carlo VaR**:
Simulate thousands of possible future scenarios based on estimated distributions.

| Confidence Level | Z-Score | Interpretation |
|------------------|---------|----------------|
| 90% | 1.282 | Expect to exceed VaR 10% of the time |
| 95% | 1.645 | Standard institutional threshold |
| 99% | 2.326 | Conservative risk management |
| 99.9% | 3.090 | Extreme tail risk |

**Limitations of VaR:**
- VaR says nothing about losses beyond the threshold
- Assumes returns are normally distributed (crypto rarely is)
- VaR is not **subadditive** - portfolio VaR may exceed sum of individual VaRs

#### Conditional VaR (CVaR / Expected Shortfall)

CVaR addresses VaR's main limitation by answering: "If things go bad, HOW bad?"

```
CVaR(α) = E[Loss | Loss > VaR(α)]

For normal distribution:
CVaR(95%) ≈ 1.28 × VaR(95%)
```

CVaR is a **coherent risk measure** (mathematically superior to VaR) because it satisfies:
- **Monotonicity**: Higher losses → Higher risk
- **Subadditivity**: Diversification reduces risk
- **Positive Homogeneity**: Scaling positions scales risk proportionally
- **Translation Invariance**: Adding cash reduces risk by that amount

#### Maximum Drawdown (MDD)

MDD measures the largest peak-to-trough decline before a new high is reached.

```
Drawdown(t) = (Peak(t) - Value(t)) / Peak(t)
MDD = max(Drawdown(t)) for all t
```

**Why MDD Matters:**
- Psychologically devastating: A 50% drawdown requires 100% gain to recover
- Historical MDD predicts future potential losses
- Used in the Calmar Ratio: Annualized Return / Max Drawdown

| Drawdown | Required Recovery |
|----------|-------------------|
| 10% | 11.1% |
| 25% | 33.3% |
| 50% | 100% |
| 75% | 300% |
| 90% | 900% |

#### Risk-Adjusted Return Metrics

**Sharpe Ratio** (Risk per unit of total volatility):
```
Sharpe = (Rp - Rf) / σp

Where:
- Rp = Portfolio return
- Rf = Risk-free rate (often 0% in crypto)
- σp = Portfolio standard deviation
```
Annualized: Multiply daily Sharpe by √252

| Sharpe Ratio | Interpretation |
|--------------|----------------|
| < 0 | Losing money |
| 0 - 0.5 | Poor |
| 0.5 - 1.0 | Acceptable |
| 1.0 - 2.0 | Good |
| > 2.0 | Excellent |
| > 3.0 | Exceptional (verify it's real!) |

**Sortino Ratio** (Penalizes only downside volatility):
```
Sortino = (Rp - Rf) / σd

Where σd = Downside deviation (std dev of negative returns only)
```
Sortino > Sharpe indicates positive skew (desirable)

**Calmar Ratio** (Return per unit of worst-case loss):
```
Calmar = Annualized Return / Maximum Drawdown
```
Particularly useful for evaluating trend-following strategies.

**Formulas Summary:**
```
VaR(95%) = Portfolio Value × z(0.95) × σ × √t
CVaR = E[Loss | Loss > VaR]
MDD = max((Peak - Trough) / Peak)
Sharpe = (Return - Rf) / σ
Sortino = (Return - Rf) / Downside σ
Calmar = Annual Return / MDD
```

### SQL: Calculate VaR and CVaR

```sql
-- Historical VaR and CVaR for each token
WITH daily_returns AS (
    SELECT
        token,
        time_bucket('1 day', "timestampUTC") AS day,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS daily_return_pct
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '180 days'
    GROUP BY token, day
),
return_stats AS (
    SELECT
        token,
        COUNT(*) AS trading_days,
        AVG(daily_return_pct) AS avg_return,
        STDDEV(daily_return_pct) AS volatility,

        -- VaR at different confidence levels (using percentile)
        PERCENTILE_CONT(0.05) WITHIN GROUP (ORDER BY daily_return_pct) AS var_95_pct,
        PERCENTILE_CONT(0.01) WITHIN GROUP (ORDER BY daily_return_pct) AS var_99_pct,

        -- For CVaR calculation
        ARRAY_AGG(daily_return_pct ORDER BY daily_return_pct) AS sorted_returns
    FROM daily_returns
    GROUP BY token
    HAVING COUNT(*) >= 60
)
SELECT
    token,
    trading_days,
    ROUND(avg_return::numeric, 3) AS avg_daily_return_pct,
    ROUND(volatility::numeric, 3) AS daily_volatility_pct,
    ROUND(volatility * SQRT(252)::numeric, 2) AS annual_volatility_pct,

    -- Historical VaR
    ROUND((-var_95_pct)::numeric, 2) AS var_95_pct,
    ROUND((-var_99_pct)::numeric, 2) AS var_99_pct,

    -- CVaR (Expected Shortfall) - average of returns below VaR
    ROUND((
        SELECT -AVG(r)
        FROM UNNEST(sorted_returns) AS r
        WHERE r < var_95_pct
    )::numeric, 2) AS cvar_95_pct,

    -- Parametric VaR (assuming normal distribution)
    ROUND((avg_return - 1.645 * volatility)::numeric, 2) AS parametric_var_95_pct,

    -- Sharpe-like ratio
    ROUND((avg_return * 252) / NULLIF(volatility * SQRT(252), 0)::numeric, 3) AS annualized_sharpe
FROM return_stats
ORDER BY var_95_pct ASC  -- Most risky first
LIMIT 30;
```

### SQL: Maximum Drawdown Analysis

```sql
-- Calculate maximum drawdown for each token
WITH price_series AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last(price, "timestampUTC") AS close_price
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY token, hour
),
running_max AS (
    SELECT
        token,
        hour,
        close_price,
        MAX(close_price) OVER (PARTITION BY token ORDER BY hour ROWS UNBOUNDED PRECEDING) AS running_peak,
        (close_price - MAX(close_price) OVER (PARTITION BY token ORDER BY hour ROWS UNBOUNDED PRECEDING)) /
            NULLIF(MAX(close_price) OVER (PARTITION BY token ORDER BY hour ROWS UNBOUNDED PRECEDING), 0) * 100 AS drawdown_pct
    FROM price_series
)
SELECT
    token,
    ROUND(MIN(drawdown_pct)::numeric, 2) AS max_drawdown_pct,
    ROUND(AVG(drawdown_pct)::numeric, 2) AS avg_drawdown_pct,

    -- Time in drawdown
    COUNT(*) FILTER (WHERE drawdown_pct < -5) AS hours_in_5pct_drawdown,
    COUNT(*) FILTER (WHERE drawdown_pct < -10) AS hours_in_10pct_drawdown,

    -- Recovery metrics
    (SELECT hour FROM running_max rm2 WHERE rm2.token = running_max.token ORDER BY drawdown_pct LIMIT 1) AS max_dd_time,

    -- Current drawdown
    (SELECT drawdown_pct FROM running_max rm2 WHERE rm2.token = running_max.token ORDER BY hour DESC LIMIT 1) AS current_drawdown_pct
FROM running_max
GROUP BY token
HAVING COUNT(*) >= 100
ORDER BY MIN(drawdown_pct) ASC
LIMIT 30;
```

### SQL: Trader-Level Risk Metrics

```sql
-- Per-trader risk analysis
WITH trader_pnl AS (
    SELECT
        address,
        DATE("timestampUTC") AS trade_date,
        SUM(
            CASE WHEN side = 'BUY' THEN -size_usd ELSE size_usd END
        ) AS daily_pnl_estimate
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY address, DATE("timestampUTC")
),
trader_stats AS (
    SELECT
        address,
        COUNT(*) AS trading_days,
        SUM(daily_pnl_estimate) AS total_pnl,
        AVG(daily_pnl_estimate) AS avg_daily_pnl,
        STDDEV(daily_pnl_estimate) AS pnl_volatility,
        MIN(daily_pnl_estimate) AS worst_day,
        MAX(daily_pnl_estimate) AS best_day,
        PERCENTILE_CONT(0.05) WITHIN GROUP (ORDER BY daily_pnl_estimate) AS var_95
    FROM trader_pnl
    GROUP BY address
    HAVING COUNT(*) >= 20
)
SELECT
    address,
    trading_days,
    ROUND(total_pnl::numeric, 2) AS total_pnl_usd,
    ROUND(avg_daily_pnl::numeric, 2) AS avg_daily_pnl_usd,
    ROUND(pnl_volatility::numeric, 2) AS daily_vol_usd,
    ROUND(worst_day::numeric, 2) AS worst_day_usd,
    ROUND(best_day::numeric, 2) AS best_day_usd,
    ROUND((-var_95)::numeric, 2) AS daily_var_95_usd,

    -- Risk-adjusted return
    ROUND((avg_daily_pnl / NULLIF(pnl_volatility, 0))::numeric, 3) AS daily_sharpe,

    -- Calmar ratio (return / max drawdown proxy)
    ROUND((total_pnl / NULLIF(ABS(worst_day), 0))::numeric, 2) AS calmar_ratio
FROM trader_stats
WHERE total_pnl > 0
ORDER BY daily_sharpe DESC
LIMIT 50;
```

---

## Part 6: Technical Indicators

### The Theory

Technical analysis is the study of market action (price, volume, time) to forecast future price direction. While controversial in academic circles (Efficient Market Hypothesis suggests it shouldn't work), technical analysis remains widely used by practitioners because:

1. **Self-Fulfilling Prophecy**: If enough traders act on the same signals, they create the predicted movement
2. **Behavioral Patterns**: Human psychology creates recurring patterns in market data
3. **Risk Management**: Even skeptics use technical levels for stop-loss and position sizing

#### Categories of Technical Indicators

| Category | Purpose | Examples |
|----------|---------|----------|
| **Trend** | Identify market direction | Moving Averages, ADX |
| **Momentum** | Measure speed of price changes | RSI, MACD, Stochastic |
| **Volatility** | Measure price variability | Bollinger Bands, ATR |
| **Volume** | Confirm price movements | OBV, VWAP |

#### Moving Averages

Moving averages smooth price data to identify trends.

**Simple Moving Average (SMA)**:
```
SMA(n) = (P₁ + P₂ + ... + Pₙ) / n
```
Equal weight to all periods.

**Exponential Moving Average (EMA)**:
```
EMA(t) = α × Price(t) + (1 - α) × EMA(t-1)

Where α = 2 / (n + 1)
```
More weight to recent prices, more responsive to changes.

**Golden Cross / Death Cross**:
- Golden Cross: Short-term MA crosses above long-term MA (bullish)
- Death Cross: Short-term MA crosses below long-term MA (bearish)
- Common pairs: 50/200 day, 12/26 EMA

#### RSI (Relative Strength Index)

RSI measures momentum on a 0-100 scale, developed by J. Welles Wilder (1978).

```
RSI = 100 - (100 / (1 + RS))

Where:
RS = Average Gain / Average Loss (over n periods, typically 14)
```

| RSI Level | Signal | Context |
|-----------|--------|---------|
| > 70 | Overbought | Consider selling/shorting |
| < 30 | Oversold | Consider buying |
| 50 | Neutral | No strong signal |
| Divergence | Price makes new high, RSI doesn't | Potential reversal |

**RSI Limitations**:
- Can stay overbought/oversold in strong trends
- Better in ranging markets than trending
- Use with other indicators for confirmation

#### MACD (Moving Average Convergence Divergence)

MACD measures the relationship between two EMAs, developed by Gerald Appel (1979).

```
MACD Line = EMA(12) - EMA(26)
Signal Line = EMA(9) of MACD Line
Histogram = MACD Line - Signal Line
```

**MACD Signals**:
- **Bullish**: MACD crosses above Signal Line
- **Bearish**: MACD crosses below Signal Line
- **Divergence**: Price trend vs MACD trend conflict (potential reversal)

#### VWAP (Volume-Weighted Average Price)

VWAP is the benchmark price based on volume and price.

```
VWAP = Σ(Price × Volume) / Σ(Volume)
```

**VWAP Usage**:
- Price above VWAP: Bullish intraday bias
- Price below VWAP: Bearish intraday bias
- Institutional traders use VWAP as execution benchmark
- Good for intraday, resets each day

#### Bollinger Bands

Bollinger Bands measure volatility and potential overbought/oversold conditions.

```
Middle Band = SMA(20)
Upper Band = SMA(20) + 2 × StdDev(20)
Lower Band = SMA(20) - 2 × StdDev(20)
```

**Bollinger Band Signals**:
- Price touching upper band: Overbought (in ranging market)
- Price touching lower band: Oversold (in ranging market)
- Band squeeze: Low volatility, potential breakout coming
- Band expansion: High volatility, trend in progress

### SQL: Moving Averages with Crossover Signals

```sql
-- Calculate SMA and EMA with crossover signals
WITH price_data AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last(price, "timestampUTC") AS close_price,
        SUM(size_usd) AS volume_usd
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, hour
),
moving_averages AS (
    SELECT
        token,
        hour,
        close_price,
        volume_usd,

        -- Simple Moving Averages
        AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 11 PRECEDING) AS sma_12,
        AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 25 PRECEDING) AS sma_26,
        AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 49 PRECEDING) AS sma_50,

        -- Volume-Weighted Average Price (VWAP)
        SUM(close_price * volume_usd) OVER (PARTITION BY token ORDER BY hour ROWS 23 PRECEDING) /
            NULLIF(SUM(volume_usd) OVER (PARTITION BY token ORDER BY hour ROWS 23 PRECEDING), 0) AS vwap_24h,

        -- Previous values for crossover detection
        LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 11 PRECEDING))
            OVER (PARTITION BY token ORDER BY hour) AS prev_sma_12,
        LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 25 PRECEDING))
            OVER (PARTITION BY token ORDER BY hour) AS prev_sma_26
    FROM price_data
)
SELECT
    token,
    hour,
    ROUND(close_price::numeric, 4) AS price,
    ROUND(sma_12::numeric, 4) AS sma_12,
    ROUND(sma_26::numeric, 4) AS sma_26,
    ROUND(sma_50::numeric, 4) AS sma_50,
    ROUND(vwap_24h::numeric, 4) AS vwap_24h,

    -- Crossover signals
    CASE
        WHEN sma_12 > sma_26 AND prev_sma_12 <= prev_sma_26 THEN 'GOLDEN_CROSS'
        WHEN sma_12 < sma_26 AND prev_sma_12 >= prev_sma_26 THEN 'DEATH_CROSS'
        ELSE NULL
    END AS crossover_signal,

    -- Price relative to MAs
    ROUND((close_price - sma_50) / NULLIF(sma_50, 0) * 100::numeric, 2) AS pct_above_sma50,

    -- VWAP signal
    CASE
        WHEN close_price > vwap_24h THEN 'ABOVE_VWAP'
        ELSE 'BELOW_VWAP'
    END AS vwap_position
FROM moving_averages
WHERE sma_50 IS NOT NULL
ORDER BY token, hour DESC
LIMIT 500;
```

### SQL: RSI (Relative Strength Index)

```sql
-- Calculate RSI with overbought/oversold signals
WITH price_changes AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last(price, "timestampUTC") AS close_price,
        last(price, "timestampUTC") - first(price, "timestampUTC") AS price_change
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, hour
),
gains_losses AS (
    SELECT
        token,
        hour,
        close_price,
        CASE WHEN price_change > 0 THEN price_change ELSE 0 END AS gain,
        CASE WHEN price_change < 0 THEN ABS(price_change) ELSE 0 END AS loss
    FROM price_changes
),
rsi_calc AS (
    SELECT
        token,
        hour,
        close_price,
        AVG(gain) OVER (PARTITION BY token ORDER BY hour ROWS 13 PRECEDING) AS avg_gain,
        AVG(loss) OVER (PARTITION BY token ORDER BY hour ROWS 13 PRECEDING) AS avg_loss
    FROM gains_losses
)
SELECT
    token,
    hour,
    ROUND(close_price::numeric, 4) AS price,
    ROUND(
        100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001)))::numeric,
        2
    ) AS rsi_14,

    -- RSI signals
    CASE
        WHEN 100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001))) > 70 THEN 'OVERBOUGHT'
        WHEN 100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001))) < 30 THEN 'OVERSOLD'
        ELSE 'NEUTRAL'
    END AS rsi_signal,

    -- RSI divergence check (price up but RSI down = bearish divergence)
    LAG(100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001))), 5)
        OVER (PARTITION BY token ORDER BY hour) AS rsi_5h_ago,
    LAG(close_price, 5) OVER (PARTITION BY token ORDER BY hour) AS price_5h_ago
FROM rsi_calc
WHERE avg_gain IS NOT NULL
ORDER BY token, hour DESC
LIMIT 500;
```

### SQL: Bollinger Bands

```sql
-- Bollinger Bands with squeeze detection
WITH price_data AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last(price, "timestampUTC") AS close_price
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '14 days'
    GROUP BY token, hour
),
bollinger AS (
    SELECT
        token,
        hour,
        close_price,
        AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 19 PRECEDING) AS sma_20,
        STDDEV(close_price) OVER (PARTITION BY token ORDER BY hour ROWS 19 PRECEDING) AS std_20
    FROM price_data
)
SELECT
    token,
    hour,
    ROUND(close_price::numeric, 4) AS price,
    ROUND(sma_20::numeric, 4) AS middle_band,
    ROUND((sma_20 + 2 * std_20)::numeric, 4) AS upper_band,
    ROUND((sma_20 - 2 * std_20)::numeric, 4) AS lower_band,

    -- Band width (squeeze indicator)
    ROUND((4 * std_20 / NULLIF(sma_20, 0) * 100)::numeric, 2) AS band_width_pct,

    -- Position within bands
    ROUND(((close_price - (sma_20 - 2 * std_20)) / NULLIF(4 * std_20, 0))::numeric, 3) AS percent_b,

    -- Signals
    CASE
        WHEN close_price > sma_20 + 2 * std_20 THEN 'ABOVE_UPPER'
        WHEN close_price < sma_20 - 2 * std_20 THEN 'BELOW_LOWER'
        WHEN 4 * std_20 / NULLIF(sma_20, 0) < 0.02 THEN 'SQUEEZE'
        ELSE 'IN_BANDS'
    END AS bb_signal
FROM bollinger
WHERE std_20 IS NOT NULL
ORDER BY token, hour DESC
LIMIT 500;
```

---

## Part 7: Time-of-Day Analysis

### The Theory

Time-based patterns are among the most robust and persistent anomalies in financial markets. Unlike many "alpha signals" that decay when discovered, time-of-day effects often persist because they're driven by fundamental market structure rather than information asymmetry.

#### Why Time Matters in Trading

1. **Institutional Constraints**: Large funds have trading windows, creating predictable volume patterns
2. **Global Time Zone Effects**: Different participant pools are active at different hours
3. **Market Microstructure**: Opening/closing auctions create systematic price patterns
4. **Human Psychology**: Traders behave differently at market open vs. close

#### The Three Trading Sessions (UTC)

| Session | UTC Hours | Key Characteristics |
|---------|-----------|---------------------|
| **Asia** | 00:00 - 08:00 | Lower volume, trend-following, reacts to US close |
| **Europe** | 08:00 - 16:00 | High volume, often sets the day's range |
| **US** | 14:00 - 22:00 | Highest volume, most volatility, drives daily close |

#### Session Overlaps - Liquidity Peaks

Session overlaps are when two major trading regions are active simultaneously:

| Overlap | UTC Hours | Why It Matters |
|---------|-----------|----------------|
| **Asia-Europe** | 07:00 - 09:00 | Asian traders closing, European traders opening |
| **Europe-US** | 14:00 - 17:00 | Highest global liquidity, tightest spreads |

**Trading Insight**: Session overlaps often produce:
- Higher volume (2-3x normal)
- Tighter bid-ask spreads
- More reliable breakouts
- Better execution for large orders

#### Day-of-Week Effects

Academic research has documented persistent day-of-week effects:

| Day | Traditional Pattern | Crypto Pattern |
|-----|---------------------|----------------|
| **Monday** | Historically negative ("Monday Effect") | Mixed - often gap fill from weekend |
| **Tuesday** | Often positive | Typically strong continuation |
| **Wednesday** | Middle of range | Mid-week reversals common |
| **Thursday** | Pre-weekend positioning | Volume picks up |
| **Friday** | Positive bias (weekend effect) | Lower volume, position reduction |
| **Weekend** | Markets closed | Crypto: Lower liquidity, potential for big moves |

#### The T-Statistic for Statistical Significance

When analyzing time-based returns, we need to know if patterns are statistically significant or just random noise:

**T-Statistic Formula:**
```
T = (Mean Return) / (Standard Error)
T = (Mean Return) / (Std Dev / √n)
```

| T-Statistic | Significance |
|-------------|--------------|
| > 2.0 | Statistically significant at 95% confidence |
| > 2.6 | Significant at 99% confidence |
| < 1.0 | Likely random noise |

#### Hour-of-Day Patterns

Specific hours often exhibit persistent characteristics:

| Hour (UTC) | Pattern | Explanation |
|------------|---------|-------------|
| 00:00-02:00 | Low volume, gradual moves | Asia session start |
| 08:00-09:00 | Spike in volatility | Europe open |
| 14:30 | Sharp move | US equity market open (NYSE/NASDAQ) |
| 16:00 | Volume spike | European close |
| 20:00-21:00 | Second wind | US afternoon trading |

#### Practical Application

**Best Hours for Different Trading Styles:**

| Style | Best Hours (UTC) | Reason |
|-------|------------------|--------|
| Scalping | 08:00-10:00, 14:00-17:00 | High liquidity, tight spreads |
| Trend Following | 14:00-20:00 | Strongest directional moves |
| Mean Reversion | 00:00-06:00 | Range-bound, lower volatility |
| Breakout Trading | 08:00, 14:30 | Session opens drive breakouts |

### SQL: Hourly Performance Patterns

```sql
-- Analyze returns by hour of day (UTC)
WITH hourly_data AS (
    SELECT
        token,
        EXTRACT(HOUR FROM "timestampUTC") AS hour_utc,
        DATE("timestampUTC") AS trade_date,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS hourly_return,
        SUM(size_usd) AS hourly_volume
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, EXTRACT(HOUR FROM "timestampUTC"), DATE("timestampUTC")
)
SELECT
    token,
    hour_utc::int,

    -- Assign trading session
    CASE
        WHEN hour_utc BETWEEN 0 AND 7 THEN 'ASIA'
        WHEN hour_utc BETWEEN 8 AND 15 THEN 'EUROPE'
        ELSE 'US'
    END AS session,

    COUNT(*) AS sample_days,
    ROUND(AVG(hourly_return)::numeric, 4) AS avg_return_pct,
    ROUND(STDDEV(hourly_return)::numeric, 4) AS volatility_pct,
    ROUND(AVG(hourly_volume)::numeric, 0) AS avg_volume_usd,

    -- Win rate for this hour
    ROUND(COUNT(*) FILTER (WHERE hourly_return > 0)::float / COUNT(*) * 100, 1) AS positive_rate_pct,

    -- Sharpe for this hour
    ROUND((AVG(hourly_return) / NULLIF(STDDEV(hourly_return), 0))::numeric, 3) AS hourly_sharpe
FROM hourly_data
GROUP BY token, hour_utc
HAVING COUNT(*) >= 20
ORDER BY token, hour_utc;
```

### SQL: Day-of-Week Effects

```sql
-- Weekly seasonality analysis
WITH daily_data AS (
    SELECT
        token,
        DATE("timestampUTC") AS trade_date,
        EXTRACT(DOW FROM "timestampUTC") AS day_of_week,  -- 0=Sunday
        TO_CHAR("timestampUTC", 'Day') AS day_name,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS daily_return,
        SUM(size_usd) AS daily_volume
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY token, DATE("timestampUTC"), EXTRACT(DOW FROM "timestampUTC"), TO_CHAR("timestampUTC", 'Day')
)
SELECT
    token,
    TRIM(day_name) AS day,
    day_of_week::int,
    COUNT(*) AS sample_weeks,

    ROUND(AVG(daily_return)::numeric, 3) AS avg_return_pct,
    ROUND(STDDEV(daily_return)::numeric, 3) AS volatility_pct,
    ROUND(AVG(daily_volume) / 1e6, 2) AS avg_volume_millions,

    -- Best/worst performance
    ROUND(MAX(daily_return)::numeric, 2) AS best_day_pct,
    ROUND(MIN(daily_return)::numeric, 2) AS worst_day_pct,

    -- T-stat for mean return
    ROUND((AVG(daily_return) / NULLIF(STDDEV(daily_return) / SQRT(COUNT(*)), 0))::numeric, 2) AS t_statistic
FROM daily_data
GROUP BY token, day_name, day_of_week
HAVING COUNT(*) >= 10
ORDER BY token, day_of_week;
```

### SQL: Session Overlap Analysis

```sql
-- Performance during session overlaps (high liquidity periods)
WITH session_periods AS (
    SELECT
        token,
        "timestampUTC",
        EXTRACT(HOUR FROM "timestampUTC") AS hour_utc,
        size_usd,
        price,
        CASE
            WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 7 AND 9 THEN 'ASIA_EUROPE_OVERLAP'
            WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 13 AND 17 THEN 'EUROPE_US_OVERLAP'
            ELSE 'SINGLE_SESSION'
        END AS session_type
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
),
session_stats AS (
    SELECT
        token,
        session_type,
        time_bucket('1 hour', "timestampUTC") AS hour,
        SUM(size_usd) AS volume,
        last(price, "timestampUTC") - first(price, "timestampUTC") AS price_change
    FROM session_periods
    GROUP BY token, session_type, hour
)
SELECT
    token,
    session_type,
    COUNT(*) AS periods,
    ROUND(AVG(volume)::numeric, 0) AS avg_hourly_volume,
    ROUND(STDDEV(price_change)::numeric, 4) AS avg_hourly_volatility,
    ROUND(AVG(ABS(price_change))::numeric, 4) AS avg_abs_move,

    -- Relative volume (vs single session)
    ROUND(AVG(volume) / NULLIF((
        SELECT AVG(volume) FROM session_stats s2
        WHERE s2.token = session_stats.token AND s2.session_type = 'SINGLE_SESSION'
    ), 0), 2) AS relative_volume
FROM session_stats
GROUP BY token, session_type
ORDER BY token, session_type;
```

---

## Part 8: Liquidation Cascade Analysis

### The Theory

Liquidation cascades are among the most violent and predictable phenomena in leveraged markets. Understanding their mechanics is crucial for both risk management and identifying trading opportunities.

#### What is a Liquidation Cascade?

A liquidation cascade is a positive feedback loop where:

```
Price Drop → Liquidations → Forced Selling → More Price Drop → More Liquidations
```

This can compress hours of normal price movement into minutes or seconds.

#### The Mechanics of Leverage and Liquidation

**Maintenance Margin Formula:**
```
Liquidation Price (Long) = Entry Price × (1 - 1/Leverage + Maintenance Margin%)
Liquidation Price (Short) = Entry Price × (1 + 1/Leverage - Maintenance Margin%)
```

**Example at 10x Leverage (2% maintenance):**
| Position | Entry | Liquidation | Distance |
|----------|-------|-------------|----------|
| Long | $100 | $92 | -8% |
| Short | $100 | $108 | +8% |

#### Why Liquidations Cluster at Certain Prices

Traders tend to use similar leverage and entry points:

| Leverage | Common Entry | Liquidation Level | Clustering |
|----------|--------------|-------------------|------------|
| 20x | Recent highs | ~5% below | High |
| 10x | Support levels | ~10% below | Medium |
| 5x | Moving averages | ~20% below | Lower |

**Round Number Effect**: Liquidations cluster around psychological levels ($50,000, $100, etc.) because:
- Traders set stop-losses at round numbers
- Leverage calculators default to round entries
- Support/resistance forms at these levels

#### Anatomy of a Cascade Event

**Phase 1: Trigger** (0-60 seconds)
- Large market sell or liquidation initiates
- Price breaks through support
- First wave of stop-losses triggered

**Phase 2: Acceleration** (1-5 minutes)
- Leveraged longs hit liquidation prices
- Forced market sells create more downward pressure
- Bid-side liquidity evaporates

**Phase 3: Climax** (variable)
- Cascade reaches maximum velocity
- Multiple liquidations per second
- Price overshoots fair value

**Phase 4: Reversal** (if any)
- Buying opportunity recognized
- Short liquidations begin (if bounce sharp)
- Price stabilizes or bounces

#### Key Metrics for Cascade Detection

| Metric | Normal | Cascade Warning | Cascade Active |
|--------|--------|-----------------|----------------|
| Liquidations/5min | < 5 | 5-15 | > 15 |
| Liq Volume (% of daily) | < 2% | 2-10% | > 10% |
| Price Impact per Liq | < 0.1% | 0.1-0.5% | > 0.5% |
| Bid Depth Ratio | > 0.8 | 0.5-0.8 | < 0.5 |

#### Trading the Cascade

**During the Cascade:**
- DO NOT try to catch the falling knife mid-cascade
- Wait for liquidation velocity to slow
- Watch for "exhaustion" signals (decreasing liq size)

**Post-Cascade Opportunities:**
1. **Bounce Play**: Enter long after cascade exhaustion
2. **Contrarian Signal**: High liquidation volume often precedes reversals
3. **Spread Capture**: Bid-ask spreads widen significantly during cascades

**Risk Management:**
- Never hold overleveraged positions near liquidation walls
- Monitor funding rates (extreme funding often precedes cascades)
- Set stop-losses AWAY from round numbers

#### Historical Cascade Events

| Event | Asset | Price Drop | Duration | Recovery Time |
|-------|-------|------------|----------|---------------|
| May 2021 Crash | BTC | -30% | 4 hours | 2 months |
| Dec 2021 Flash | BTC | -21% | 1 hour | 3 days |
| Nov 2022 FTX | ETH | -25% | 2 days | 6 months |
| Aug 2024 Carry | BTC | -18% | 6 hours | 1 week |

### SQL: Liquidation Clustering Detection

```sql
-- Detect liquidation cascades
WITH liquidation_data AS (
    SELECT
        token,
        "timestampUTC",
        liq_size_usd,
        price,

        -- Count liquidations in rolling window
        COUNT(*) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
        ) AS liqs_5min,

        SUM(liq_size_usd) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
        ) AS liq_volume_5min
    FROM hl_liquidations
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
),
cascades AS (
    SELECT
        token,
        "timestampUTC" AS cascade_start,
        liqs_5min,
        liq_volume_5min,

        -- Identify cascade events (5+ liquidations in 5 min)
        CASE WHEN liqs_5min >= 5 THEN TRUE ELSE FALSE END AS is_cascade,

        -- Price impact
        (price - LAG(price, 5) OVER (PARTITION BY token ORDER BY "timestampUTC")) /
            NULLIF(LAG(price, 5) OVER (PARTITION BY token ORDER BY "timestampUTC"), 0) * 100 AS price_impact_pct
    FROM liquidation_data
)
SELECT
    token,
    DATE(cascade_start) AS cascade_date,
    COUNT(*) FILTER (WHERE is_cascade) AS cascade_events,
    SUM(liq_volume_5min) FILTER (WHERE is_cascade) AS cascade_volume_usd,
    MAX(liqs_5min) AS max_liqs_in_5min,

    -- Average price impact during cascades
    ROUND(AVG(price_impact_pct) FILTER (WHERE is_cascade)::numeric, 2) AS avg_cascade_impact_pct,
    ROUND(MIN(price_impact_pct) FILTER (WHERE is_cascade)::numeric, 2) AS max_cascade_drop_pct
FROM cascades
GROUP BY token, DATE(cascade_start)
HAVING COUNT(*) FILTER (WHERE is_cascade) > 0
ORDER BY cascade_volume_usd DESC
LIMIT 50;
```

### SQL: Liquidation Level Analysis

```sql
-- Identify price levels with high liquidation concentration
WITH liquidation_prices AS (
    SELECT
        token,
        ROUND(price::numeric, 2) AS price_level,
        COUNT(*) AS liq_count,
        SUM(liq_size_usd) AS total_liq_volume,
        AVG(liq_size_usd) AS avg_liq_size,
        MIN("timestampUTC") AS first_liq,
        MAX("timestampUTC") AS last_liq
    FROM hl_liquidations
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, ROUND(price::numeric, 2)
),
current_prices AS (
    SELECT DISTINCT ON (token)
        token,
        price AS current_price
    FROM hl_trades
    ORDER BY token, "timestampUTC" DESC
)
SELECT
    lp.token,
    lp.price_level,
    lp.liq_count,
    ROUND(lp.total_liq_volume::numeric, 0) AS total_volume_usd,
    ROUND(lp.avg_liq_size::numeric, 0) AS avg_size_usd,

    -- Distance from current price
    ROUND(((lp.price_level - cp.current_price) / cp.current_price * 100)::numeric, 2) AS pct_from_current,

    -- Time since last liquidation at this level
    EXTRACT(EPOCH FROM (NOW() - lp.last_liq)) / 3600 AS hours_since_last
FROM liquidation_prices lp
JOIN current_prices cp ON lp.token = cp.token
WHERE lp.liq_count >= 3
ORDER BY lp.token, lp.total_liq_volume DESC
LIMIT 100;
```

### SQL: Liquidation as Contrarian Signal

```sql
-- Test if high liquidation volume predicts reversals
WITH daily_liqs AS (
    SELECT
        token,
        DATE("timestampUTC") AS trade_date,
        SUM(liq_size_usd) AS daily_liq_volume,
        COUNT(*) AS liq_count
    FROM hl_liquidations
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY token, DATE("timestampUTC")
),
daily_returns AS (
    SELECT
        token,
        DATE("timestampUTC") AS trade_date,
        (last(price, "timestampUTC") - first(price, "timestampUTC")) /
            NULLIF(first(price, "timestampUTC"), 0) * 100 AS daily_return
    FROM hl_trades
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY token, DATE("timestampUTC")
),
combined AS (
    SELECT
        r.token,
        r.trade_date,
        COALESCE(l.daily_liq_volume, 0) AS liq_volume,
        r.daily_return,
        LEAD(r.daily_return) OVER (PARTITION BY r.token ORDER BY r.trade_date) AS next_day_return,
        -- Percentile rank of liquidation volume
        PERCENT_RANK() OVER (PARTITION BY r.token ORDER BY COALESCE(l.daily_liq_volume, 0)) AS liq_percentile
    FROM daily_returns r
    LEFT JOIN daily_liqs l ON r.token = l.token AND r.trade_date = l.trade_date
)
SELECT
    token,

    -- Returns following high liquidation days (top 10%)
    ROUND(AVG(next_day_return) FILTER (WHERE liq_percentile > 0.9)::numeric, 3) AS return_after_high_liq,
    COUNT(*) FILTER (WHERE liq_percentile > 0.9) AS high_liq_days,

    -- Returns following low liquidation days
    ROUND(AVG(next_day_return) FILTER (WHERE liq_percentile < 0.1)::numeric, 3) AS return_after_low_liq,

    -- Difference (contrarian signal strength)
    ROUND((AVG(next_day_return) FILTER (WHERE liq_percentile > 0.9) -
           AVG(next_day_return) FILTER (WHERE liq_percentile < 0.1))::numeric, 3) AS contrarian_edge
FROM combined
WHERE next_day_return IS NOT NULL
GROUP BY token
HAVING COUNT(*) FILTER (WHERE liq_percentile > 0.9) >= 5
ORDER BY contrarian_edge DESC
LIMIT 20;
```

---

## Part 9: Funding Rate Arbitrage

### The Theory

Funding rates are the mechanism that keeps perpetual futures prices anchored to spot prices. Unlike traditional futures that expire, perpetuals require periodic payments between longs and shorts to prevent divergence.

#### How Funding Rates Work

```
When Perp Price > Spot Price:
  → Market is bullish (more demand for longs)
  → POSITIVE funding rate
  → Longs PAY shorts every 8 hours
  → This encourages shorting, pushing perp price down toward spot

When Perp Price < Spot Price:
  → Market is bearish (more demand for shorts)
  → NEGATIVE funding rate
  → Shorts PAY longs every 8 hours
  → This encourages longing, pushing perp price up toward spot
```

#### Funding Rate Formula (Hyperliquid)

**Standard Funding Calculation:**
```
Funding Rate = (Premium Index) + clamp(Interest Rate - Premium Index, -0.05%, 0.05%)

Where:
- Premium Index = (Perp Price - Spot Price) / Spot Price
- Interest Rate = 0.01% per 8 hours (typical)
```

**Payment Calculation:**
```
Funding Payment = Position Size × Funding Rate
```

#### Annualizing Funding Rates

Since funding is paid every 8 hours (3 times per day):

| Funding Rate | Daily Rate | Annual Rate |
|--------------|------------|-------------|
| 0.01% | 0.03% | ~11% |
| 0.05% | 0.15% | ~55% |
| 0.10% | 0.30% | ~110% |
| -0.02% | -0.06% | ~-22% |

**Annualization Formula:**
```
Annual Rate = Funding Rate × 3 × 365 × 100%
```

#### Arbitrage Strategies

**Strategy 1: Cash-and-Carry (Positive Funding)**
```
When funding is extremely positive:
1. BUY spot (or hold USDC equivalent)
2. SHORT perpetual at same size
3. Collect funding payments from longs
4. Net exposure = 0 (delta neutral)
5. Profit = Funding payments - Fees

Risk: Funding can flip negative; liquidation risk if undercollateralized
```

**Strategy 2: Reverse Cash-and-Carry (Negative Funding)**
```
When funding is extremely negative:
1. SELL/SHORT spot (if possible) or borrow
2. LONG perpetual at same size
3. Collect funding payments from shorts
4. Net exposure = 0 (delta neutral)

Risk: Harder to execute; borrowing costs may exceed funding
```

**Strategy 3: Funding Rate Mean Reversion**
```
When funding hits extreme Z-scores (>2 or <-2):
1. Bet that funding will revert toward 0
2. Position opposite to the crowded side
3. Exit when funding normalizes

This is NOT delta-neutral - you're taking directional risk
```

#### Key Metrics and Thresholds

| Metric | Low | Normal | High | Extreme |
|--------|-----|--------|------|---------|
| Funding Rate (8h) | < -0.05% | -0.01% to 0.03% | 0.03% to 0.10% | > 0.10% |
| Annualized | < -55% | -11% to 33% | 33% to 110% | > 110% |
| Z-Score | < -2 | -1 to 1 | 1 to 2 | > 2 |

#### Cross-Venue Funding Arbitrage

When funding rates diverge across exchanges:

| Scenario | Action |
|----------|--------|
| Hyperliquid funding > Binance funding | Short on Hyperliquid, Long on Binance |
| Hyperliquid funding < dYdX funding | Long on Hyperliquid, Short on dYdX |

**Profit = Higher Funding Received - Lower Funding Paid - Transfer Costs**

#### Historical Funding Patterns

| Market Condition | Typical Funding | Duration |
|------------------|-----------------|----------|
| Bull market / FOMO | +0.05% to +0.30% | Days to weeks |
| Bear market / Panic | -0.05% to -0.20% | Hours to days |
| Sideways / Range | -0.01% to +0.02% | Weeks to months |
| Local top formation | Extreme positive spike | 1-3 days |
| Local bottom formation | Extreme negative spike | Hours |

### SQL: Funding Rate Extremes

```sql
-- Identify extreme funding rate opportunities
WITH funding_data AS (
    SELECT
        token,
        "timestampUTC",
        funding_rate,
        -- Annualized funding rate
        funding_rate * 3 * 365 * 100 AS annualized_pct
    FROM hl_funding
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
),
funding_stats AS (
    SELECT
        token,
        AVG(funding_rate) AS avg_funding,
        STDDEV(funding_rate) AS funding_std,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY funding_rate) AS funding_95,
        PERCENTILE_CONT(0.05) WITHIN GROUP (ORDER BY funding_rate) AS funding_05
    FROM funding_data
    GROUP BY token
),
current_funding AS (
    SELECT DISTINCT ON (token)
        token,
        funding_rate,
        "timestampUTC"
    FROM hl_funding
    ORDER BY token, "timestampUTC" DESC
)
SELECT
    cf.token,
    ROUND((cf.funding_rate * 100)::numeric, 4) AS current_funding_pct,
    ROUND((cf.funding_rate * 3 * 365 * 100)::numeric, 2) AS annualized_pct,
    ROUND((fs.avg_funding * 100)::numeric, 4) AS avg_funding_pct,

    -- Z-score (how many std devs from mean)
    ROUND(((cf.funding_rate - fs.avg_funding) / NULLIF(fs.funding_std, 0))::numeric, 2) AS z_score,

    -- Percentile
    ROUND((cf.funding_rate - fs.funding_05) / NULLIF(fs.funding_95 - fs.funding_05, 0) * 100, 1) AS percentile,

    -- Signal
    CASE
        WHEN cf.funding_rate > fs.funding_95 THEN 'EXTREME_LONG_CROWDED'
        WHEN cf.funding_rate < fs.funding_05 THEN 'EXTREME_SHORT_CROWDED'
        WHEN ABS(cf.funding_rate) < fs.funding_std / 2 THEN 'NEUTRAL'
        ELSE 'MODERATE'
    END AS funding_signal,

    -- Estimated cost/profit of holding (per day)
    ROUND((cf.funding_rate * 3 * 100)::numeric, 4) AS daily_cost_pct
FROM current_funding cf
JOIN funding_stats fs ON cf.token = fs.token
ORDER BY ABS(z_score) DESC
LIMIT 30;
```

### SQL: Funding Rate Mean Reversion

```sql
-- Test funding rate mean reversion as a trading signal
WITH funding_with_future AS (
    SELECT
        token,
        "timestampUTC",
        funding_rate,
        -- Future price returns (8h forward, next funding period)
        LEAD(funding_rate) OVER (PARTITION BY token ORDER BY "timestampUTC") AS next_funding,

        -- Z-score at this point
        (funding_rate - AVG(funding_rate) OVER (PARTITION BY token ORDER BY "timestampUTC" ROWS 30 PRECEDING)) /
            NULLIF(STDDEV(funding_rate) OVER (PARTITION BY token ORDER BY "timestampUTC" ROWS 30 PRECEDING), 0) AS zscore
    FROM hl_funding
    WHERE "timestampUTC" >= NOW() - INTERVAL '60 days'
),
signal_analysis AS (
    SELECT
        token,
        CASE
            WHEN zscore > 2 THEN 'HIGH'
            WHEN zscore < -2 THEN 'LOW'
            ELSE 'NORMAL'
        END AS funding_regime,
        funding_rate,
        next_funding,
        next_funding - funding_rate AS funding_change
    FROM funding_with_future
    WHERE next_funding IS NOT NULL
)
SELECT
    token,
    funding_regime,
    COUNT(*) AS occurrences,

    -- Does extreme funding revert?
    ROUND(AVG(funding_change)::numeric * 10000, 4) AS avg_funding_change_bps,
    ROUND(AVG(ABS(funding_change))::numeric * 10000, 4) AS avg_abs_change_bps,

    -- What % of time does it revert toward mean?
    ROUND(COUNT(*) FILTER (
        WHERE (funding_regime = 'HIGH' AND funding_change < 0) OR
              (funding_regime = 'LOW' AND funding_change > 0)
    )::float / COUNT(*) * 100, 1) AS reversion_rate_pct
FROM signal_analysis
GROUP BY token, funding_regime
HAVING COUNT(*) >= 10
ORDER BY token, funding_regime;
```

---

## Part 10: Trader Segmentation & Clustering

### The Theory

Understanding different trader types is crucial for market analysis. Traders naturally cluster into distinct behavioral archetypes based on their trading patterns, and identifying these groups enables alpha generation through copy-trading, contrarian signals, and market regime detection.

#### Why Trader Segmentation Matters

```
1. ALPHA DISCOVERY
   - Follow consistently profitable traders
   - Fade systematically losing traders
   - Detect smart money vs retail flow

2. MARKET STRUCTURE
   - Understand who provides/takes liquidity
   - Identify market makers vs directional traders
   - Detect institutional vs retail participation

3. RISK SIGNALS
   - Crowded trades (everyone positioned same way)
   - Capitulation signals (mass exits)
   - FOMO signals (late entry clusters)
```

#### The Four Main Trader Archetypes

| Type | Trade Frequency | Hold Time | Avg Trade Size | Win Rate | Key Characteristics |
|------|-----------------|-----------|----------------|----------|---------------------|
| **Scalper** | Very High (100+/day) | Seconds to minutes | Small | 55-65% | Profits from bid-ask spread, volume-focused |
| **Day Trader** | Medium (5-20/day) | Minutes to hours | Medium | 45-55% | Closes positions same day, momentum plays |
| **Swing Trader** | Low (1-5/week) | Hours to days | Large | 40-50% | Catches multi-day moves, trend following |
| **Position Trader** | Very Low (1-5/month) | Days to weeks | Very Large | 35-45% | Macro thesis, conviction trades |

#### Key Metrics for Trader Clustering

**1. Activity Metrics:**
```
Trades per Day = Total Trades / Active Days
Volume per Trade = Total Volume / Trade Count
Active Hours = Distinct hours with trades
```

**2. Timing Metrics:**
```
Average Hold Time = Σ(Close Time - Open Time) / Trade Count
Turnover Rate = Volume / Average Position Size
```

**3. Performance Metrics:**
```
Win Rate = Profitable Trades / Total Trades
Profit Factor = Gross Profit / Gross Loss
Average P&L per Trade = Net P&L / Trade Count
```

**4. Risk Metrics:**
```
Average Leverage = Σ(Position Size / Margin) / Trade Count
Max Drawdown = Largest peak-to-trough decline
Sharpe Ratio = (Return - Risk-Free) / Volatility
```

#### Clustering Methodology

**Feature Normalization (Z-Score):**
```
Z = (Value - Mean) / Standard Deviation

This transforms all metrics to comparable scales.
```

**K-Means Clustering:**
```
1. Select K (number of clusters, typically 4-6)
2. Initialize K centroids randomly
3. Assign each trader to nearest centroid
4. Recalculate centroids as cluster means
5. Repeat steps 3-4 until convergence
```

**Cluster Interpretation:**
| Cluster | High Values | Low Values | Likely Type |
|---------|-------------|------------|-------------|
| A | Frequency, Win Rate | Hold Time, Size | Scalper |
| B | Hold Time, Size | Frequency | Position Trader |
| C | Volume, Leverage | Win Rate | Aggressive Speculator |
| D | Win Rate, Sharpe | Drawdown, Leverage | Disciplined Pro |

#### Trader Quality Scoring

**Composite Score Formula:**
```
Score = w1×(Win Rate) + w2×(Sharpe) + w3×(Profit Factor) + w4×(Consistency)

Where:
- w1 = 0.25 (win rate weight)
- w2 = 0.30 (risk-adjusted return weight)
- w3 = 0.25 (profit factor weight)
- w4 = 0.20 (consistency weight)
```

**Consistency Metric:**
```
Consistency = 1 - (Std Dev of Daily Returns / Mean Daily Return)
```

#### Following the Smart Money

| Metric | Smart Money | Retail | How to Detect |
|--------|-------------|--------|---------------|
| Position Timing | Early in trends | Late in trends | Entry relative to price extremes |
| Sizing | Scales in/out | All-in/All-out | Trade count vs position size variance |
| Risk Management | Stops losses | Holds losers | Average loss vs average win |
| Market Timing | Contrarian at extremes | Follows crowd | Position vs sentiment/funding |

#### Practical Applications

**1. Copy-Trading Filter:**
```
Good candidates:
- Sharpe Ratio > 1.5
- Win Rate > 50%
- Trade Count > 100 (statistical significance)
- Max Drawdown < 30%
- Active > 30 days (not lucky streak)
```

**2. Contrarian Signal:**
```
When retail traders are:
- 80%+ positioned one direction → Consider opposite
- Heavily leveraged at extremes → Expect liquidation cascade
- All entering at same time → Top/bottom may be near
```

**3. Market Regime Detection:**
```
High Scalper Activity → Range-bound, mean-reverting
High Swing Trader Activity → Trending market
High Position Trader Entries → Major inflection point
```

### SQL: Trader Behavior Metrics

```sql
-- Calculate comprehensive trader behavior metrics
WITH trader_trades AS (
    SELECT
        address,
        "timestampUTC",
        token,
        side,
        size_usd,
        price,

        -- Time between trades
        EXTRACT(EPOCH FROM "timestampUTC" - LAG("timestampUTC") OVER (PARTITION BY address ORDER BY "timestampUTC")) AS seconds_between_trades
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
),
trader_metrics AS (
    SELECT
        address,
        COUNT(*) AS total_trades,
        COUNT(DISTINCT DATE("timestampUTC")) AS active_days,
        COUNT(DISTINCT token) AS tokens_traded,
        SUM(size_usd) AS total_volume,
        AVG(size_usd) AS avg_trade_size,
        STDDEV(size_usd) AS trade_size_std,

        -- Frequency metrics
        AVG(seconds_between_trades) AS avg_seconds_between,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY seconds_between_trades) AS median_seconds_between,

        -- Directional metrics
        SUM(CASE WHEN side = 'BUY' THEN 1 ELSE 0 END)::float / COUNT(*) AS buy_ratio,

        -- Session concentration
        COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM "timestampUTC") BETWEEN 0 AND 7) AS asia_trades,
        COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM "timestampUTC") BETWEEN 8 AND 15) AS europe_trades,
        COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM "timestampUTC") BETWEEN 16 AND 23) AS us_trades
    FROM trader_trades
    WHERE seconds_between_trades IS NOT NULL
    GROUP BY address
    HAVING COUNT(*) >= 50
)
SELECT
    address,
    total_trades,
    active_days,
    tokens_traded,
    ROUND((total_volume / 1e6)::numeric, 2) AS volume_millions,
    ROUND(avg_trade_size::numeric, 2) AS avg_size_usd,

    -- Trader type classification
    CASE
        WHEN avg_seconds_between < 60 AND total_trades > 1000 THEN 'SCALPER'
        WHEN avg_seconds_between < 300 AND total_trades > 500 THEN 'DAY_TRADER'
        WHEN avg_seconds_between < 3600 THEN 'ACTIVE_TRADER'
        WHEN avg_seconds_between < 86400 THEN 'SWING_TRADER'
        ELSE 'POSITION_TRADER'
    END AS trader_type,

    -- Size consistency (lower = more consistent)
    ROUND((trade_size_std / NULLIF(avg_trade_size, 0))::numeric, 2) AS size_consistency,

    -- Directional bias
    CASE
        WHEN buy_ratio > 0.6 THEN 'LONG_BIAS'
        WHEN buy_ratio < 0.4 THEN 'SHORT_BIAS'
        ELSE 'BALANCED'
    END AS direction_bias,

    -- Session preference
    CASE
        WHEN asia_trades > europe_trades AND asia_trades > us_trades THEN 'ASIA'
        WHEN europe_trades > us_trades THEN 'EUROPE'
        ELSE 'US'
    END AS primary_session,

    ROUND(buy_ratio::numeric, 3) AS buy_ratio,
    ROUND((avg_seconds_between / 60)::numeric, 1) AS avg_minutes_between_trades
FROM trader_metrics
ORDER BY total_volume DESC
LIMIT 500;
```

### SQL: Identify Top Performers by Trader Type

```sql
-- Find best performers in each trader category
WITH trader_performance AS (
    SELECT
        address,
        COUNT(*) AS trades,
        SUM(size_usd) AS volume,
        AVG(EXTRACT(EPOCH FROM "timestampUTC" - LAG("timestampUTC") OVER (PARTITION BY address ORDER BY "timestampUTC"))) AS avg_time_between,

        -- Simplified P&L estimate
        SUM(CASE
            WHEN side = 'BUY' THEN -size_usd
            ELSE size_usd
        END) AS net_flow
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY address
    HAVING COUNT(*) >= 30
),
classified AS (
    SELECT
        *,
        CASE
            WHEN avg_time_between < 60 THEN 'SCALPER'
            WHEN avg_time_between < 300 THEN 'DAY_TRADER'
            WHEN avg_time_between < 3600 THEN 'ACTIVE_TRADER'
            ELSE 'SWING_TRADER'
        END AS trader_type
    FROM trader_performance
    WHERE avg_time_between IS NOT NULL
)
SELECT
    trader_type,
    COUNT(*) AS trader_count,

    -- Performance distribution
    ROUND(AVG(net_flow)::numeric, 2) AS avg_net_flow,
    ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY net_flow)::numeric, 2) AS top_10pct_flow,
    ROUND(PERCENTILE_CONT(0.1) WITHIN GROUP (ORDER BY net_flow)::numeric, 2) AS bottom_10pct_flow,

    -- Activity metrics
    ROUND(AVG(trades)::numeric, 0) AS avg_trades,
    ROUND(AVG(volume / 1e6)::numeric, 2) AS avg_volume_millions,

    -- Win rate proxy
    ROUND(COUNT(*) FILTER (WHERE net_flow > 0)::float / COUNT(*) * 100, 1) AS positive_flow_pct
FROM classified
GROUP BY trader_type
ORDER BY avg_net_flow DESC;
```

### SQL: Similar Trader Discovery

```sql
-- Find traders with similar behavior patterns
WITH trader_features AS (
    SELECT
        address,
        COUNT(*) AS trades,
        COUNT(DISTINCT token) AS tokens,
        AVG(size_usd) AS avg_size,
        STDDEV(size_usd) / NULLIF(AVG(size_usd), 0) AS size_cv,
        SUM(CASE WHEN side = 'BUY' THEN 1 ELSE 0 END)::float / COUNT(*) AS buy_ratio,
        COUNT(DISTINCT DATE("timestampUTC")) AS active_days
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY address
    HAVING COUNT(*) >= 100
),
normalized AS (
    SELECT
        address,
        trades,
        -- Normalize features to 0-1 scale
        (trades - MIN(trades) OVER()) / NULLIF(MAX(trades) OVER() - MIN(trades) OVER(), 0) AS n_trades,
        (tokens - MIN(tokens) OVER()) / NULLIF(MAX(tokens) OVER() - MIN(tokens) OVER(), 0) AS n_tokens,
        (avg_size - MIN(avg_size) OVER()) / NULLIF(MAX(avg_size) OVER() - MIN(avg_size) OVER(), 0) AS n_size,
        (size_cv - MIN(size_cv) OVER()) / NULLIF(MAX(size_cv) OVER() - MIN(size_cv) OVER(), 0) AS n_cv,
        buy_ratio AS n_buy_ratio,
        (active_days - MIN(active_days) OVER()) / NULLIF(MAX(active_days) OVER() - MIN(active_days) OVER(), 0) AS n_days
    FROM trader_features
)
-- Find similar traders to a target address
SELECT
    n2.address AS similar_trader,
    n1.address AS target_trader,
    ROUND(SQRT(
        POWER(n1.n_trades - n2.n_trades, 2) +
        POWER(n1.n_tokens - n2.n_tokens, 2) +
        POWER(n1.n_size - n2.n_size, 2) +
        POWER(n1.n_cv - n2.n_cv, 2) +
        POWER(n1.n_buy_ratio - n2.n_buy_ratio, 2) +
        POWER(n1.n_days - n2.n_days, 2)
    )::numeric, 4) AS euclidean_distance
FROM normalized n1
CROSS JOIN normalized n2
WHERE n1.address != n2.address
  AND n1.trades > 500  -- Target high-activity traders
ORDER BY n1.trades DESC, euclidean_distance ASC
LIMIT 100;
```

---

## Part 11: Orderbook Depth Analysis

### The Theory

Orderbook depth analysis is fundamental to understanding market microstructure and liquidity conditions. The orderbook represents the complete picture of supply and demand at any given moment, revealing where large players are positioned and how the market might react to incoming orders.

#### Why Orderbook Depth Matters

The orderbook provides critical insights that price data alone cannot:

1. **Liquidity Assessment**: How much size can be executed at various price levels without significant slippage
2. **Support/Resistance Identification**: Where large orders create natural barriers
3. **Market Sentiment**: Bid/ask imbalances reveal directional bias
4. **Execution Planning**: Optimal order sizing based on available liquidity
5. **Manipulation Detection**: Spoofing patterns and phantom liquidity

#### Key Orderbook Concepts

| Metric | Description | Trading Application |
|--------|-------------|---------------------|
| **Bid-Ask Spread** | Difference between best bid and ask | Measure of liquidity cost |
| **Depth at X%** | Total liquidity within X% of mid-price | Slippage estimation |
| **Bid/Ask Imbalance** | Ratio of bid to ask liquidity | Short-term directional signal |
| **Price Impact** | How much price moves for a given order size | Position sizing |
| **Liquidity Walls** | Concentrated orders at specific levels | S/R identification |

#### Liquidity Imbalance Formula

```
Bid/Ask Imbalance = (Total Bid Volume - Total Ask Volume) / (Total Bid Volume + Total Ask Volume)
```

Values range from -1 (all asks) to +1 (all bids). Values > 0.3 or < -0.3 indicate significant imbalance.

#### Slippage Estimation

Expected slippage for order size S:
```
Slippage = Σ(Volume_i × Price_i) / Σ(Volume_i) - Mid_Price
```
Where i represents each price level consumed by the order.

### SQL: Liquidity Depth Over Time

```sql
-- Analyze how liquidity at various depth levels changes over time
SELECT
    time_bucket('1 hour', "timestampUTC") AS hour,
    token,

    -- Average liquidity at different depth levels
    ROUND(AVG(total_bid_usdc_1pct + total_ask_usdc_1pct)::numeric, 0) AS liquidity_1pct,
    ROUND(AVG(total_bid_usdc_2_5pct + total_ask_usdc_2_5pct)::numeric, 0) AS liquidity_2_5pct,
    ROUND(AVG(total_bid_usdc_5pct + total_ask_usdc_5pct)::numeric, 0) AS liquidity_5pct,
    ROUND(AVG(total_bid_usdc_10pct + total_ask_usdc_10pct)::numeric, 0) AS liquidity_10pct,

    -- Net imbalance (positive = more bids, bullish)
    ROUND(AVG(net_usdc_1pct)::numeric, 0) AS net_imbalance_1pct,
    ROUND(AVG(net_usdc_5pct)::numeric, 0) AS net_imbalance_5pct,

    -- Imbalance ratio (-1 to 1)
    ROUND(AVG(net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0))::numeric, 3) AS imbalance_ratio_1pct

FROM hl_orderbook
WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
  AND token = 'BTC'
GROUP BY hour, token
ORDER BY hour DESC
LIMIT 168;  -- One week of hourly data
```

### SQL: Bid-Ask Imbalance Signal

```sql
-- Calculate bid/ask imbalance as a trading signal
-- Strong imbalance often precedes price movement in that direction
WITH orderbook_snapshots AS (
    SELECT
        "timestampUTC",
        token,
        total_bid_usdc_1pct,
        total_ask_usdc_1pct,
        net_usdc_1pct,

        -- Calculate imbalance ratio
        CASE
            WHEN (total_bid_usdc_1pct + total_ask_usdc_1pct) > 0
            THEN net_usdc_1pct / (total_bid_usdc_1pct + total_ask_usdc_1pct)
            ELSE 0
        END AS imbalance_ratio,

        -- Classify imbalance strength
        CASE
            WHEN net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0) > 0.4 THEN 'STRONG_BID'
            WHEN net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0) > 0.2 THEN 'MODERATE_BID'
            WHEN net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0) < -0.4 THEN 'STRONG_ASK'
            WHEN net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0) < -0.2 THEN 'MODERATE_ASK'
            ELSE 'BALANCED'
        END AS imbalance_signal

    FROM hl_orderbook
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
      AND token IN ('BTC', 'ETH', 'SOL')
)
SELECT
    token,
    imbalance_signal,
    COUNT(*) AS occurrences,
    ROUND(AVG(imbalance_ratio)::numeric, 3) AS avg_imbalance,
    ROUND(MIN(imbalance_ratio)::numeric, 3) AS min_imbalance,
    ROUND(MAX(imbalance_ratio)::numeric, 3) AS max_imbalance
FROM orderbook_snapshots
GROUP BY token, imbalance_signal
ORDER BY token, occurrences DESC;
```

### SQL: Liquidity Wall Detection

```sql
-- Identify significant liquidity concentrations (walls)
-- These often act as support/resistance levels
WITH liquidity_levels AS (
    SELECT
        "timestampUTC",
        token,

        -- Check for walls at different liquidity thresholds
        px_10k_liquidity_bid,
        px_10k_liquidity_ask,
        px_50k_liquidity_bid,
        px_50k_liquidity_ask,
        px_100k_liquidity_bid,
        px_100k_liquidity_ask,
        px_250k_liquidity_bid,
        px_250k_liquidity_ask,

        -- Calculate spread between liquidity levels
        (px_100k_liquidity_ask - px_100k_liquidity_bid) AS spread_100k,
        (px_250k_liquidity_ask - px_250k_liquidity_bid) AS spread_250k

    FROM hl_orderbook
    WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
      AND token = 'BTC'
)
SELECT
    time_bucket('5 minutes', "timestampUTC") AS bucket,
    token,

    -- Average price levels where $100k liquidity exists
    ROUND(AVG(px_100k_liquidity_bid)::numeric, 2) AS support_100k,
    ROUND(AVG(px_100k_liquidity_ask)::numeric, 2) AS resistance_100k,

    -- Average price levels where $250k liquidity exists
    ROUND(AVG(px_250k_liquidity_bid)::numeric, 2) AS support_250k,
    ROUND(AVG(px_250k_liquidity_ask)::numeric, 2) AS resistance_250k,

    -- Spread metrics
    ROUND(AVG(spread_100k)::numeric, 2) AS avg_spread_100k,
    ROUND(AVG(spread_250k)::numeric, 2) AS avg_spread_250k

FROM liquidity_levels
GROUP BY bucket, token
ORDER BY bucket DESC;
```

### SQL: Slippage Estimation

```sql
-- Estimate expected slippage for different order sizes
SELECT
    time_bucket('15 minutes', "timestampUTC") AS bucket,
    token,

    -- Liquidity available at each depth level (in USD)
    ROUND(AVG(total_bid_usdc_0_5pct)::numeric, 0) AS bid_liq_0_5pct,
    ROUND(AVG(total_bid_usdc_1pct)::numeric, 0) AS bid_liq_1pct,
    ROUND(AVG(total_bid_usdc_2_5pct)::numeric, 0) AS bid_liq_2_5pct,
    ROUND(AVG(total_bid_usdc_5pct)::numeric, 0) AS bid_liq_5pct,

    -- Slippage estimate: for $100k sell order
    CASE
        WHEN AVG(total_bid_usdc_0_5pct) >= 100000 THEN 0.25  -- Less than 0.5% slippage
        WHEN AVG(total_bid_usdc_1pct) >= 100000 THEN 0.75   -- Between 0.5-1% slippage
        WHEN AVG(total_bid_usdc_2_5pct) >= 100000 THEN 1.75 -- Between 1-2.5% slippage
        WHEN AVG(total_bid_usdc_5pct) >= 100000 THEN 3.75   -- Between 2.5-5% slippage
        ELSE 7.5  -- More than 5% slippage
    END AS estimated_slippage_pct_100k_sell,

    -- Slippage estimate: for $500k sell order
    CASE
        WHEN AVG(total_bid_usdc_0_5pct) >= 500000 THEN 0.25
        WHEN AVG(total_bid_usdc_1pct) >= 500000 THEN 0.75
        WHEN AVG(total_bid_usdc_2_5pct) >= 500000 THEN 1.75
        WHEN AVG(total_bid_usdc_5pct) >= 500000 THEN 3.75
        ELSE 7.5
    END AS estimated_slippage_pct_500k_sell

FROM hl_orderbook
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
  AND token IN ('BTC', 'ETH', 'SOL')
GROUP BY bucket, token
ORDER BY bucket DESC, token
LIMIT 100;
```

### SQL: Liquidity Anomaly Detection

```sql
-- Detect unusual changes in orderbook depth
-- Sudden liquidity drops can signal incoming volatility
WITH liquidity_stats AS (
    SELECT
        token,
        "timestampUTC",
        (total_bid_usdc_5pct + total_ask_usdc_5pct) AS total_liquidity,
        AVG(total_bid_usdc_5pct + total_ask_usdc_5pct) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING
        ) AS avg_liquidity_1h,
        STDDEV(total_bid_usdc_5pct + total_ask_usdc_5pct) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING
        ) AS stddev_liquidity_1h
    FROM hl_orderbook
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
)
SELECT
    token,
    "timestampUTC",
    ROUND(total_liquidity::numeric, 0) AS current_liquidity,
    ROUND(avg_liquidity_1h::numeric, 0) AS avg_liquidity,
    ROUND(((total_liquidity - avg_liquidity_1h) / NULLIF(stddev_liquidity_1h, 0))::numeric, 2) AS z_score,
    CASE
        WHEN (total_liquidity - avg_liquidity_1h) / NULLIF(stddev_liquidity_1h, 0) < -2 THEN 'LIQUIDITY_CRISIS'
        WHEN (total_liquidity - avg_liquidity_1h) / NULLIF(stddev_liquidity_1h, 0) < -1.5 THEN 'LOW_LIQUIDITY'
        WHEN (total_liquidity - avg_liquidity_1h) / NULLIF(stddev_liquidity_1h, 0) > 2 THEN 'LIQUIDITY_SURGE'
        ELSE 'NORMAL'
    END AS liquidity_status
FROM liquidity_stats
WHERE ABS((total_liquidity - avg_liquidity_1h) / NULLIF(stddev_liquidity_1h, 0)) > 1.5
ORDER BY "timestampUTC" DESC
LIMIT 50;
```

---

## Part 12: Open Interest Analytics

### The Theory

Open Interest (OI) represents the total number of outstanding derivative contracts (perpetual futures positions) that have not been settled. Unlike trading volume which measures activity, OI measures commitment - traders who have skin in the game.

#### Why Open Interest Matters

Open Interest provides crucial signals that complement price and volume analysis:

1. **Trend Confirmation**: Rising OI with price confirms trend strength
2. **Position Crowding**: Extreme OI signals potential squeeze scenarios
3. **Leverage Assessment**: OI/Market Cap ratio indicates leverage in the system
4. **Liquidation Risk**: High OI at certain prices increases cascade risk
5. **Smart Money Tracking**: Large position changes reveal institutional activity

#### OI Interpretation Framework

| Price Movement | OI Change | Interpretation |
|---------------|-----------|----------------|
| Price Up | OI Up | Strong bullish - new longs entering |
| Price Up | OI Down | Weak rally - short covering |
| Price Down | OI Up | Strong bearish - new shorts entering |
| Price Down | OI Down | Weak decline - long liquidation |

#### Position Concentration Risk

When too many traders are on the same side:
```
Crowding Score = (Long OI - Short OI) / Total OI
```
Extreme values (>0.3 or <-0.3) often precede mean reversion.

#### Key Metrics

- **Position Value**: szi × entry_px (total notional exposure)
- **Unrealized PnL**: Current profit/loss on open positions
- **Margin Utilization**: margin_used / account_value (liquidation risk)
- **Funding Burden**: cum_funding_since_open (carrying cost)

### SQL: Aggregate Open Interest by Token

```sql
-- Calculate total open interest and position metrics per token
SELECT
    token,

    -- Position counts
    COUNT(*) AS total_positions,
    COUNT(*) FILTER (WHERE szi > 0) AS long_positions,
    COUNT(*) FILTER (WHERE szi < 0) AS short_positions,

    -- Total notional open interest (absolute value)
    ROUND(SUM(ABS(position_value))::numeric, 0) AS total_oi_usd,
    ROUND(SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END)::numeric, 0) AS long_oi_usd,
    ROUND(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)::numeric, 0) AS short_oi_usd,

    -- Long/Short ratio
    ROUND(
        SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
        NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0)::numeric,
    2) AS long_short_ratio,

    -- Average position size
    ROUND(AVG(ABS(position_value))::numeric, 0) AS avg_position_size,

    -- Unrealized PnL totals
    ROUND(SUM(unrealized_pnl)::numeric, 0) AS total_unrealized_pnl,
    ROUND(SUM(CASE WHEN szi > 0 THEN unrealized_pnl ELSE 0 END)::numeric, 0) AS longs_unrealized_pnl,
    ROUND(SUM(CASE WHEN szi < 0 THEN unrealized_pnl ELSE 0 END)::numeric, 0) AS shorts_unrealized_pnl

FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY token
ORDER BY total_oi_usd DESC
LIMIT 20;
```

### SQL: Position Crowding Analysis

```sql
-- Identify crowded trades where one side dominates
-- Crowded positions are susceptible to squeezes
WITH position_summary AS (
    SELECT
        token,
        SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) AS long_value,
        SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END) AS short_value,
        SUM(ABS(position_value)) AS total_oi,
        COUNT(*) AS position_count
    FROM perp_asset_positions_v2
    WHERE timestamp >= NOW() - INTERVAL '1 hour'
    GROUP BY token
    HAVING SUM(ABS(position_value)) > 1000000  -- Min $1M OI
)
SELECT
    token,
    ROUND(long_value::numeric, 0) AS long_oi,
    ROUND(short_value::numeric, 0) AS short_oi,
    ROUND(total_oi::numeric, 0) AS total_oi,

    -- Crowding score: -1 (all shorts) to +1 (all longs)
    ROUND(((long_value - short_value) / NULLIF(total_oi, 0))::numeric, 3) AS crowding_score,

    -- Risk assessment
    CASE
        WHEN (long_value - short_value) / NULLIF(total_oi, 0) > 0.4 THEN 'SHORT_SQUEEZE_RISK'
        WHEN (long_value - short_value) / NULLIF(total_oi, 0) > 0.25 THEN 'LONG_CROWDED'
        WHEN (long_value - short_value) / NULLIF(total_oi, 0) < -0.4 THEN 'LONG_SQUEEZE_RISK'
        WHEN (long_value - short_value) / NULLIF(total_oi, 0) < -0.25 THEN 'SHORT_CROWDED'
        ELSE 'BALANCED'
    END AS crowding_status,

    position_count

FROM position_summary
ORDER BY ABS((long_value - short_value) / NULLIF(total_oi, 0)) DESC;
```

### SQL: Large Position Holders (Whale Tracking)

```sql
-- Identify whales and their positioning
WITH whale_positions AS (
    SELECT
        wallet_id,
        token,
        szi,
        entry_px,
        position_value,
        unrealized_pnl,
        liquidation_px,
        margin_used,
        cum_funding_since_open,
        CASE WHEN szi > 0 THEN 'LONG' ELSE 'SHORT' END AS direction
    FROM perp_asset_positions_v2
    WHERE timestamp >= NOW() - INTERVAL '1 hour'
      AND ABS(position_value) > 100000  -- Positions > $100k
)
SELECT
    token,
    direction,
    COUNT(*) AS whale_count,
    ROUND(SUM(ABS(position_value))::numeric, 0) AS total_whale_oi,
    ROUND(AVG(ABS(position_value))::numeric, 0) AS avg_whale_position,
    ROUND(MAX(ABS(position_value))::numeric, 0) AS largest_position,
    ROUND(SUM(unrealized_pnl)::numeric, 0) AS total_whale_pnl,
    ROUND(AVG(unrealized_pnl / NULLIF(ABS(position_value), 0) * 100)::numeric, 2) AS avg_pnl_pct,
    ROUND(SUM(cum_funding_since_open)::numeric, 0) AS total_funding_paid
FROM whale_positions
GROUP BY token, direction
ORDER BY total_whale_oi DESC
LIMIT 30;
```

### SQL: Position Entry Analysis

```sql
-- Analyze where positions were entered (potential S/R levels)
SELECT
    token,

    -- Entry price distribution
    ROUND(percentile_cont(0.25) WITHIN GROUP (ORDER BY entry_px)::numeric, 2) AS entry_p25,
    ROUND(percentile_cont(0.50) WITHIN GROUP (ORDER BY entry_px)::numeric, 2) AS entry_median,
    ROUND(percentile_cont(0.75) WITHIN GROUP (ORDER BY entry_px)::numeric, 2) AS entry_p75,

    -- Weighted average entry (by position size)
    ROUND((SUM(entry_px * ABS(position_value)) / NULLIF(SUM(ABS(position_value)), 0))::numeric, 2) AS weighted_avg_entry,

    -- Entry price range
    ROUND(MIN(entry_px)::numeric, 2) AS min_entry,
    ROUND(MAX(entry_px)::numeric, 2) AS max_entry,

    -- Position count
    COUNT(*) AS positions

FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
  AND token IN ('BTC', 'ETH', 'SOL')
GROUP BY token
ORDER BY positions DESC;
```

### SQL: Liquidation Risk Assessment

```sql
-- Identify positions at risk of liquidation
WITH position_risk AS (
    SELECT
        token,
        wallet_id,
        szi,
        entry_px,
        position_value,
        liquidation_px,
        margin_used,

        -- Distance to liquidation (as percentage)
        CASE
            WHEN szi > 0 THEN (entry_px - liquidation_px) / NULLIF(entry_px, 0) * 100  -- Longs
            ELSE (liquidation_px - entry_px) / NULLIF(entry_px, 0) * 100  -- Shorts
        END AS distance_to_liq_pct

    FROM perp_asset_positions_v2
    WHERE timestamp >= NOW() - INTERVAL '1 hour'
      AND liquidation_px IS NOT NULL
      AND liquidation_px > 0
)
SELECT
    token,

    -- Positions within different risk zones
    COUNT(*) FILTER (WHERE distance_to_liq_pct < 5) AS positions_within_5pct,
    COUNT(*) FILTER (WHERE distance_to_liq_pct BETWEEN 5 AND 10) AS positions_5_to_10pct,
    COUNT(*) FILTER (WHERE distance_to_liq_pct BETWEEN 10 AND 20) AS positions_10_to_20pct,
    COUNT(*) FILTER (WHERE distance_to_liq_pct > 20) AS positions_over_20pct,

    -- Value at risk (positions within 10% of liquidation)
    ROUND(SUM(ABS(position_value)) FILTER (WHERE distance_to_liq_pct < 10)::numeric, 0) AS value_at_risk_10pct,

    -- Average distance to liquidation
    ROUND(AVG(distance_to_liq_pct)::numeric, 2) AS avg_distance_to_liq_pct,

    -- Total positions analyzed
    COUNT(*) AS total_positions

FROM position_risk
GROUP BY token
ORDER BY value_at_risk_10pct DESC NULLS LAST
LIMIT 20;
```

### SQL: Open Interest Change Tracking

```sql
-- Track OI changes over time for trend confirmation
WITH hourly_oi AS (
    SELECT
        time_bucket('1 hour', timestamp) AS hour,
        token,
        SUM(ABS(position_value)) AS total_oi,
        SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) AS long_oi,
        SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END) AS short_oi,
        COUNT(*) AS position_count
    FROM perp_asset_positions_v2
    WHERE timestamp >= NOW() - INTERVAL '7 days'
      AND token IN ('BTC', 'ETH', 'SOL')
    GROUP BY hour, token
)
SELECT
    hour,
    token,
    ROUND(total_oi::numeric, 0) AS total_oi,

    -- OI change from previous hour
    ROUND((total_oi - LAG(total_oi) OVER (PARTITION BY token ORDER BY hour))::numeric, 0) AS oi_change,

    -- Percentage change
    ROUND(((total_oi - LAG(total_oi) OVER (PARTITION BY token ORDER BY hour)) /
           NULLIF(LAG(total_oi) OVER (PARTITION BY token ORDER BY hour), 0) * 100)::numeric, 2) AS oi_change_pct,

    -- Long/Short changes
    ROUND((long_oi - LAG(long_oi) OVER (PARTITION BY token ORDER BY hour))::numeric, 0) AS long_oi_change,
    ROUND((short_oi - LAG(short_oi) OVER (PARTITION BY token ORDER BY hour))::numeric, 0) AS short_oi_change,

    position_count

FROM hourly_oi
ORDER BY token, hour DESC
LIMIT 100;
```

---

## Part 13: Capital Flow Analysis

### The Theory

Capital flows - deposits and withdrawals - provide a window into aggregate trader behavior and market sentiment. Unlike price movements which can be driven by leverage and speculation, capital flows represent real money moving into and out of the exchange.

#### Why Capital Flows Matter

Tracking deposits and withdrawals reveals:

1. **Exchange Health**: Net inflows/outflows indicate platform trust
2. **Buying Power**: Fresh deposits signal dry powder entering
3. **Profit Taking**: Large withdrawals after rallies confirm distribution
4. **Whale Activity**: Large single transactions reveal institutional moves
5. **Leading Indicator**: Flow changes often precede price moves

#### Flow Analysis Framework

| Market Phase | Typical Flow Pattern |
|-------------|---------------------|
| **Accumulation** | Steady deposits, low withdrawals |
| **Markup** | Accelerating deposits as FOMO builds |
| **Distribution** | Rising withdrawals, slowing deposits |
| **Markdown** | Mass withdrawals, panic outflows |

#### Key Metrics

- **Net Flow**: Deposits - Withdrawals (positive = bullish)
- **Flow Ratio**: Deposits / Withdrawals (>1 = bullish)
- **Unique Depositors**: Breadth of buying interest
- **Avg Deposit Size**: Retail vs institutional activity
- **Withdrawal Completion Rate**: Platform health indicator

### SQL: Net Capital Flow Analysis

```sql
-- Calculate net capital flows over time
WITH deposits_agg AS (
    SELECT
        time_bucket('1 day', "timestampUTC") AS day,
        COUNT(*) AS deposit_count,
        COUNT(DISTINCT address) AS unique_depositors,
        SUM(usdc) AS total_deposits,
        AVG(usdc) AS avg_deposit,
        MAX(usdc) AS max_deposit
    FROM hl_deposits
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY day
),
withdrawals_agg AS (
    SELECT
        time_bucket('1 day', "timestampUTC") AS day,
        COUNT(*) AS withdrawal_count,
        COUNT(DISTINCT address) AS unique_withdrawers,
        SUM(amount) AS total_withdrawals,
        AVG(amount) AS avg_withdrawal,
        MAX(amount) AS max_withdrawal,
        COUNT(*) FILTER (WHERE is_finalized) AS finalized_count
    FROM hl_withdrawals
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY day
)
SELECT
    COALESCE(d.day, w.day) AS day,

    -- Deposit metrics
    COALESCE(d.deposit_count, 0) AS deposits,
    COALESCE(d.unique_depositors, 0) AS unique_depositors,
    ROUND(COALESCE(d.total_deposits, 0)::numeric, 0) AS total_deposited,
    ROUND(COALESCE(d.avg_deposit, 0)::numeric, 2) AS avg_deposit,

    -- Withdrawal metrics
    COALESCE(w.withdrawal_count, 0) AS withdrawals,
    COALESCE(w.unique_withdrawers, 0) AS unique_withdrawers,
    ROUND(COALESCE(w.total_withdrawals, 0)::numeric, 0) AS total_withdrawn,
    ROUND(COALESCE(w.avg_withdrawal, 0)::numeric, 2) AS avg_withdrawal,

    -- Net flow (positive = net inflow)
    ROUND((COALESCE(d.total_deposits, 0) - COALESCE(w.total_withdrawals, 0))::numeric, 0) AS net_flow,

    -- Flow ratio (>1 = more deposits than withdrawals)
    ROUND((COALESCE(d.total_deposits, 0) / NULLIF(COALESCE(w.total_withdrawals, 0), 0))::numeric, 2) AS flow_ratio

FROM deposits_agg d
FULL OUTER JOIN withdrawals_agg w ON d.day = w.day
ORDER BY day DESC;
```

### SQL: Large Flow Detection (Whale Watching)

```sql
-- Identify large deposits and withdrawals that might signal whale activity
WITH large_deposits AS (
    SELECT
        "timestampUTC",
        'DEPOSIT' AS flow_type,
        address,
        usdc AS amount,
        hash
    FROM hl_deposits
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
      AND usdc > 100000  -- Large deposits > $100k
),
large_withdrawals AS (
    SELECT
        "timestampUTC",
        'WITHDRAWAL' AS flow_type,
        address,
        amount,
        hash
    FROM hl_withdrawals
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
      AND amount > 100000  -- Large withdrawals > $100k
)
SELECT
    "timestampUTC",
    flow_type,
    address,
    ROUND(amount::numeric, 0) AS amount_usd,
    CASE
        WHEN amount > 1000000 THEN 'WHALE'
        WHEN amount > 500000 THEN 'LARGE'
        ELSE 'SIGNIFICANT'
    END AS size_category
FROM (
    SELECT * FROM large_deposits
    UNION ALL
    SELECT * FROM large_withdrawals
) combined
ORDER BY "timestampUTC" DESC
LIMIT 100;
```

### SQL: Hourly Flow Momentum

```sql
-- Track hourly flow momentum for short-term signals
WITH hourly_flows AS (
    SELECT
        time_bucket('1 hour', "timestampUTC") AS hour,
        SUM(usdc) AS deposits
    FROM hl_deposits
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY hour
),
hourly_withdrawals AS (
    SELECT
        time_bucket('1 hour', "timestampUTC") AS hour,
        SUM(amount) AS withdrawals
    FROM hl_withdrawals
    WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
    GROUP BY hour
),
combined AS (
    SELECT
        COALESCE(d.hour, w.hour) AS hour,
        COALESCE(d.deposits, 0) AS deposits,
        COALESCE(w.withdrawals, 0) AS withdrawals,
        COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0) AS net_flow
    FROM hourly_flows d
    FULL OUTER JOIN hourly_withdrawals w ON d.hour = w.hour
)
SELECT
    hour,
    ROUND(deposits::numeric, 0) AS deposits,
    ROUND(withdrawals::numeric, 0) AS withdrawals,
    ROUND(net_flow::numeric, 0) AS net_flow,

    -- 4-hour rolling average net flow
    ROUND(AVG(net_flow) OVER (ORDER BY hour ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)::numeric, 0) AS net_flow_4h_avg,

    -- 24-hour rolling sum
    ROUND(SUM(net_flow) OVER (ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW)::numeric, 0) AS net_flow_24h,

    -- Flow momentum signal
    CASE
        WHEN AVG(net_flow) OVER (ORDER BY hour ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) > 100000 THEN 'STRONG_INFLOW'
        WHEN AVG(net_flow) OVER (ORDER BY hour ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) > 0 THEN 'MILD_INFLOW'
        WHEN AVG(net_flow) OVER (ORDER BY hour ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) > -100000 THEN 'MILD_OUTFLOW'
        ELSE 'STRONG_OUTFLOW'
    END AS flow_signal

FROM combined
ORDER BY hour DESC
LIMIT 168;  -- 1 week of hourly data
```

### SQL: Depositor Behavior Analysis

```sql
-- Analyze deposit behavior patterns
WITH depositor_stats AS (
    SELECT
        address,
        COUNT(*) AS deposit_count,
        SUM(usdc) AS total_deposited,
        AVG(usdc) AS avg_deposit,
        MIN("timestampUTC") AS first_deposit,
        MAX("timestampUTC") AS last_deposit,
        MAX("timestampUTC") - MIN("timestampUTC") AS deposit_span
    FROM hl_deposits
    WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY address
)
SELECT
    -- Depositor classification
    CASE
        WHEN total_deposited > 1000000 THEN 'WHALE'
        WHEN total_deposited > 100000 THEN 'LARGE'
        WHEN total_deposited > 10000 THEN 'MEDIUM'
        ELSE 'SMALL'
    END AS depositor_tier,

    COUNT(*) AS depositor_count,
    ROUND(SUM(total_deposited)::numeric, 0) AS tier_total_deposited,
    ROUND(AVG(total_deposited)::numeric, 0) AS avg_total_per_depositor,
    ROUND(AVG(deposit_count)::numeric, 1) AS avg_deposits_per_user,
    ROUND(AVG(avg_deposit)::numeric, 0) AS avg_single_deposit

FROM depositor_stats
GROUP BY
    CASE
        WHEN total_deposited > 1000000 THEN 'WHALE'
        WHEN total_deposited > 100000 THEN 'LARGE'
        WHEN total_deposited > 10000 THEN 'MEDIUM'
        ELSE 'SMALL'
    END
ORDER BY tier_total_deposited DESC;
```

### SQL: New vs Returning Depositors

```sql
-- Track new user onboarding vs returning users
WITH first_deposits AS (
    SELECT
        address,
        MIN("timestampUTC") AS first_deposit_time
    FROM hl_deposits
    GROUP BY address
),
daily_deposits AS (
    SELECT
        time_bucket('1 day', d."timestampUTC") AS day,
        d.address,
        d.usdc,
        CASE
            WHEN time_bucket('1 day', d."timestampUTC") = time_bucket('1 day', f.first_deposit_time)
            THEN 'NEW'
            ELSE 'RETURNING'
        END AS depositor_type
    FROM hl_deposits d
    JOIN first_deposits f ON d.address = f.address
    WHERE d."timestampUTC" >= NOW() - INTERVAL '30 days'
)
SELECT
    day,

    -- New depositors
    COUNT(*) FILTER (WHERE depositor_type = 'NEW') AS new_depositors,
    ROUND(SUM(usdc) FILTER (WHERE depositor_type = 'NEW')::numeric, 0) AS new_depositor_volume,

    -- Returning depositors
    COUNT(*) FILTER (WHERE depositor_type = 'RETURNING') AS returning_depositors,
    ROUND(SUM(usdc) FILTER (WHERE depositor_type = 'RETURNING')::numeric, 0) AS returning_depositor_volume,

    -- Ratio metrics
    ROUND(
        COUNT(*) FILTER (WHERE depositor_type = 'NEW')::float /
        NULLIF(COUNT(*), 0) * 100,
    1) AS new_depositor_pct,

    -- Total
    COUNT(*) AS total_deposits,
    ROUND(SUM(usdc)::numeric, 0) AS total_volume

FROM daily_deposits
GROUP BY day
ORDER BY day DESC;
```

---

## Part 14: Using Pre-built CAGGs

### The Theory

Continuous Aggregates (CAGGs) are TimescaleDB's solution for real-time analytics at scale. They automatically maintain pre-computed aggregations as new data arrives, enabling sub-second query performance on billions of rows.

#### Why Use CAGGs?

1. **Performance**: Queries that would scan billions of rows complete in milliseconds
2. **Real-time**: Automatically updated as new data arrives
3. **Storage Efficient**: Store aggregates instead of raw data for historical analysis
4. **Consistent**: Materialized results ensure reproducible analytics

#### Available Pre-built CAGGs

Your database includes 131 continuous aggregates covering key use cases:

| Category | CAGGs | Purpose |
|----------|-------|---------|
| **Funding Rates** | `funding_rate_1m`, `funding_rate_1h`, `funding_rate_8h`, `funding_rate_1y` | Multi-timeframe funding analysis |
| **Token Prices** | `perp_tokens_30min`, `perp_tokens_1h`, `perp_tokens_1d` | Price aggregations |
| **Volume** | `hl_trades_volume_1min`, `hl_trades_volume_5min`, `hl_trades_volume_30min` | Trade volume metrics |
| **Orderbook** | `hl_orderbook_30min` | Liquidity aggregations |
| **Positions** | `perp_asset_positions_v2_30min`, `perp_asset_positions_v2_1h` | Position aggregations |
| **Liquidations** | `hl_liquidation_orders_stats_30min`, `hl_liquidation_heatmap_10min` | Liquidation metrics |
| **User Activity** | `hl_user_fills_count_30min`, `hl_user_fills_pnl_3h` | Trader activity |
| **Capital Flows** | `hl_c_deposits_stats_30min`, `hl_c_withdrawals_stats_30min` | Flow aggregations |

### SQL: Using Funding Rate CAGGs

```sql
-- Compare funding rates across multiple timeframes using pre-built CAGGs
-- This query would take minutes on raw data, but runs instantly on CAGGs

-- 8-hour funding rates (standard funding period)
SELECT
    bucket,
    token,
    avg_funding_rate,
    ROUND((avg_funding_rate * 3 * 365 * 100)::numeric, 2) AS annualized_rate_pct
FROM funding_rate_8h
WHERE bucket >= NOW() - INTERVAL '7 days'
  AND token IN ('BTC', 'ETH', 'SOL')
ORDER BY bucket DESC, token
LIMIT 30;
```

### SQL: Using Volume CAGGs for Analysis

```sql
-- Analyze trading volume patterns using pre-computed aggregates
SELECT
    bucket,
    token,
    trade_count,
    ROUND(total_volume::numeric, 0) AS volume_usd,
    ROUND(avg_trade_size::numeric, 2) AS avg_trade_size,
    ROUND(buy_volume::numeric, 0) AS buy_volume,
    ROUND(sell_volume::numeric, 0) AS sell_volume,

    -- Buy/Sell ratio
    ROUND((buy_volume / NULLIF(sell_volume, 0))::numeric, 2) AS buy_sell_ratio

FROM hl_trades_volume_30min
WHERE bucket >= NOW() - INTERVAL '24 hours'
  AND token IN ('BTC', 'ETH')
ORDER BY bucket DESC, token
LIMIT 100;
```

### SQL: Combining Multiple CAGGs

```sql
-- Combine price, volume, and funding data for comprehensive analysis
-- Each CAGG is pre-computed, making this join extremely fast
WITH price_data AS (
    SELECT
        bucket,
        token,
        close_price,
        high_price,
        low_price,
        (close_price - open_price) / NULLIF(open_price, 0) * 100 AS price_change_pct
    FROM perp_tokens_1h
    WHERE bucket >= NOW() - INTERVAL '24 hours'
),
volume_data AS (
    SELECT
        time_bucket('1 hour', bucket) AS bucket,
        token,
        SUM(total_volume) AS volume,
        SUM(trade_count) AS trades
    FROM hl_trades_volume_30min
    WHERE bucket >= NOW() - INTERVAL '24 hours'
    GROUP BY time_bucket('1 hour', bucket), token
),
funding_data AS (
    SELECT
        bucket,
        token,
        avg_funding_rate
    FROM funding_rate_1h
    WHERE bucket >= NOW() - INTERVAL '24 hours'
)
SELECT
    p.bucket,
    p.token,
    ROUND(p.close_price::numeric, 2) AS price,
    ROUND(p.price_change_pct::numeric, 2) AS price_change_pct,
    ROUND(v.volume::numeric, 0) AS volume,
    v.trades,
    ROUND((f.avg_funding_rate * 100)::numeric, 4) AS funding_rate_pct,

    -- Composite signal
    CASE
        WHEN p.price_change_pct > 1 AND v.volume > 1000000 AND f.avg_funding_rate > 0.0001
        THEN 'BULLISH_CONFIRMATION'
        WHEN p.price_change_pct < -1 AND v.volume > 1000000 AND f.avg_funding_rate < -0.0001
        THEN 'BEARISH_CONFIRMATION'
        ELSE 'MIXED'
    END AS signal

FROM price_data p
LEFT JOIN volume_data v ON p.bucket = v.bucket AND p.token = v.token
LEFT JOIN funding_data f ON p.bucket = f.bucket AND p.token = f.token
WHERE p.token IN ('BTC', 'ETH', 'SOL')
ORDER BY p.bucket DESC, p.token
LIMIT 100;
```

### SQL: Using Position CAGGs for OI Tracking

```sql
-- Track open interest over time using pre-aggregated position data
SELECT
    bucket,
    token,
    ROUND(total_long_value::numeric, 0) AS long_oi,
    ROUND(total_short_value::numeric, 0) AS short_oi,
    ROUND((total_long_value + total_short_value)::numeric, 0) AS total_oi,
    position_count,

    -- Imbalance
    ROUND(((total_long_value - total_short_value) /
           NULLIF(total_long_value + total_short_value, 0) * 100)::numeric, 2) AS imbalance_pct

FROM perp_asset_positions_v2_1h
WHERE bucket >= NOW() - INTERVAL '7 days'
  AND token IN ('BTC', 'ETH')
ORDER BY bucket DESC, token
LIMIT 100;
```

### SQL: Liquidation Analysis with CAGGs

```sql
-- Analyze liquidation patterns using pre-computed stats
SELECT
    bucket,
    token,
    liquidation_count,
    ROUND(total_liquidation_value::numeric, 0) AS liquidation_volume,
    long_liquidations,
    short_liquidations,
    ROUND(avg_liquidation_size::numeric, 0) AS avg_liq_size,
    ROUND(max_liquidation_size::numeric, 0) AS max_liq_size,

    -- Liquidation intensity
    CASE
        WHEN liquidation_count > 50 THEN 'HIGH_INTENSITY'
        WHEN liquidation_count > 20 THEN 'MODERATE'
        ELSE 'LOW'
    END AS liq_intensity

FROM hl_liquidation_orders_stats_30min
WHERE bucket >= NOW() - INTERVAL '7 days'
  AND token IN ('BTC', 'ETH', 'SOL')
ORDER BY bucket DESC, total_liquidation_value DESC NULLS LAST
LIMIT 100;
```

### SQL: User Activity Metrics

```sql
-- Track trader activity metrics using user fills CAGGs
SELECT
    bucket,

    -- Activity counts
    active_traders,
    total_trades,
    ROUND((total_trades::float / NULLIF(active_traders, 0))::numeric, 1) AS trades_per_trader,

    -- Volume metrics
    ROUND(total_volume::numeric, 0) AS total_volume,
    ROUND((total_volume / NULLIF(active_traders, 0))::numeric, 0) AS volume_per_trader,

    -- New vs returning
    new_traders,
    ROUND((new_traders::float / NULLIF(active_traders, 0) * 100)::numeric, 1) AS new_trader_pct

FROM hl_user_fills_count_30min
WHERE bucket >= NOW() - INTERVAL '7 days'
ORDER BY bucket DESC
LIMIT 100;
```

### SQL: Capital Flow CAGGs

```sql
-- Monitor capital flows using pre-computed deposit/withdrawal stats
WITH deposit_stats AS (
    SELECT
        bucket,
        deposit_count,
        unique_depositors,
        total_deposits,
        avg_deposit,
        max_deposit
    FROM hl_c_deposits_stats_30min
    WHERE bucket >= NOW() - INTERVAL '7 days'
),
withdrawal_stats AS (
    SELECT
        bucket,
        withdrawal_count,
        unique_withdrawers,
        total_withdrawals,
        avg_withdrawal
    FROM hl_c_withdrawals_stats_30min
    WHERE bucket >= NOW() - INTERVAL '7 days'
)
SELECT
    COALESCE(d.bucket, w.bucket) AS bucket,

    -- Deposits
    COALESCE(d.deposit_count, 0) AS deposits,
    ROUND(COALESCE(d.total_deposits, 0)::numeric, 0) AS deposit_volume,

    -- Withdrawals
    COALESCE(w.withdrawal_count, 0) AS withdrawals,
    ROUND(COALESCE(w.total_withdrawals, 0)::numeric, 0) AS withdrawal_volume,

    -- Net flow
    ROUND((COALESCE(d.total_deposits, 0) - COALESCE(w.total_withdrawals, 0))::numeric, 0) AS net_flow,

    -- Flow ratio
    ROUND((COALESCE(d.total_deposits, 0) / NULLIF(COALESCE(w.total_withdrawals, 0), 0))::numeric, 2) AS flow_ratio

FROM deposit_stats d
FULL OUTER JOIN withdrawal_stats w ON d.bucket = w.bucket
ORDER BY bucket DESC
LIMIT 100;
```

### Performance Comparison: Raw vs CAGG

```sql
-- Example: Compare query time on raw data vs CAGG
-- Raw query (would scan millions of rows):
EXPLAIN ANALYZE
SELECT
    time_bucket('1 hour', "timestampUTC") AS hour,
    token,
    AVG(funding_rate) AS avg_funding
FROM hl_funding
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY hour, token;

-- CAGG query (pre-computed, instant):
EXPLAIN ANALYZE
SELECT bucket, token, avg_funding_rate
FROM funding_rate_1h
WHERE bucket >= NOW() - INTERVAL '30 days';
```

---

## CAGG/Materialized View Recommendations

Based on the analysis of your data tables and existing CAGGs, here are recommendations:

### Already Available (No Action Needed)

Your database already has comprehensive CAGGs covering:
- Funding rates (multiple timeframes)
- Token prices (30min, 1h, 1d, etc.)
- Trading volume (1min to daily)
- Orderbook depth (30min)
- Positions (30min, 1h, 1d)
- Liquidations (10min, 30min)
- User activity (30min, 3h)
- Capital flows (30min)

### Potential New CAGGs to Consider

If you find yourself running these queries frequently, consider creating CAGGs:

1. **Whale Activity CAGG** - Track large trades/positions:
```sql
-- Example: Whale activity aggregation
CREATE MATERIALIZED VIEW whale_activity_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', "timestampUTC") AS bucket,
    token,
    COUNT(*) FILTER (WHERE size_usd > 100000) AS whale_trades,
    SUM(size_usd) FILTER (WHERE size_usd > 100000) AS whale_volume,
    AVG(size_usd) FILTER (WHERE size_usd > 100000) AS avg_whale_size
FROM hl_user_fills
GROUP BY bucket, token
WITH NO DATA;
```

2. **Bid-Ask Imbalance CAGG** - Pre-compute orderbook imbalance:
```sql
-- Example: Orderbook imbalance aggregation
CREATE MATERIALIZED VIEW orderbook_imbalance_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', "timestampUTC") AS bucket,
    token,
    AVG(net_usdc_1pct / NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0)) AS imbalance_ratio,
    AVG(total_bid_usdc_1pct + total_ask_usdc_1pct) AS total_liquidity_1pct
FROM hl_orderbook
GROUP BY bucket, token
WITH NO DATA;
```

3. **Position Crowding CAGG** - Track long/short imbalance:
```sql
-- Example: Position crowding aggregation
CREATE MATERIALIZED VIEW position_crowding_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', timestamp) AS bucket,
    token,
    SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) AS long_oi,
    SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END) AS short_oi,
    COUNT(*) AS position_count
FROM perp_asset_positions_v2
GROUP BY bucket, token
WITH NO DATA;
```

### When to Create New CAGGs vs Query Raw Data

| Scenario | Recommendation |
|----------|---------------|
| Ad-hoc analysis | Query raw data |
| Dashboard with refresh < 5 min | Create CAGG |
| Query runs > 10 seconds | Consider CAGG |
| Historical analysis (months of data) | Use existing CAGGs |
| Real-time alerting | Create CAGG with small bucket |

---

## Appendix: CAGG Optimization Audit for Parts 1-10

This section documents which queries in Parts 1-10 of this playbook could be optimized by using existing Continuous Aggregates (CAGGs) instead of raw tables.

### Audit Summary Table

| Part | Topic | Raw Tables Used | Recommended CAGGs | Impact |
|------|-------|-----------------|-------------------|--------|
| 1 | Backtesting | `hl_trades`, `hl_user_fills` | `hl_trades_volume_30min`, `perp_tokens_30min` | MEDIUM |
| 2 | Whale Watching | `hl_user_fills`, `hl_trades` | `traders_30min` | MEDIUM |
| 3 | Copy-Trading | `hl_user_fills` | Needs raw data for correlation | LOW |
| 4 | Microstructure | `hl_trades` | `hl_trades_volume_30min` (partial) | LOW |
| 5 | Risk Metrics | `hl_trades`, `hl_user_fills` | `perp_tokens_30min` for returns | MEDIUM |
| 6 | Technical Indicators | `hl_trades` | `perp_tokens_30min`, `perp_tokens_1h` | HIGH |
| 7 | Time-of-Day | `hl_trades` | `hl_trades_volume_30min` | MEDIUM |
| 8 | Liquidations | `hl_liquidations` | `hl_liquidation_orders_stats_30min` | HIGH |
| 9 | Funding Rate | `hl_funding` | `funding_rate_8h`, `funding_rate_1h` | HIGH |
| 10 | Trader Segmentation | `hl_user_fills`, `hl_trades` | `traders_30min` | HIGH |

### HIGH Impact Optimizations

These queries scan millions of rows and would benefit most from using CAGGs:

#### Part 9: Funding Rate Arbitrage

**Original (Raw Table):**
```sql
-- Scans entire hl_funding table
SELECT time_bucket('8 hours', "timestampUTC") AS bucket,
       token, AVG(funding_rate) AS avg_funding
FROM hl_funding
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY bucket, token;
```

**Optimized (Using CAGG):**
```sql
-- Uses pre-computed funding_rate_8h CAGG
SELECT bucket, token, avg_funding_rate
FROM funding_rate_8h
WHERE bucket >= NOW() - INTERVAL '30 days';
-- 100x+ faster for 30-day queries
```

#### Part 6: Technical Indicators (OHLCV)

**Original (Raw Table):**
```sql
-- Computes OHLCV from billions of trades
SELECT time_bucket('1 hour', "timestampUTC") AS bucket,
       token,
       FIRST(price, "timestampUTC") AS open,
       MAX(price) AS high,
       MIN(price) AS low,
       LAST(price, "timestampUTC") AS close,
       SUM(size_usd) AS volume
FROM hl_trades
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY bucket, token;
```

**Optimized (Using CAGG):**
```sql
-- Uses pre-computed perp_tokens_1h or perp_tokens_30min CAGG
SELECT bucket, token, open_price, high_price, low_price, close_price, volume
FROM perp_tokens_1h
WHERE bucket >= NOW() - INTERVAL '30 days';
-- Instant results vs minutes of computation
```

#### Part 8: Liquidation Analysis

**Original (Raw Table):**
```sql
-- Scans entire liquidations table
SELECT time_bucket('30 minutes', "timestampUTC") AS bucket,
       token,
       COUNT(*) AS liq_count,
       SUM(abs_size * price) AS liq_volume
FROM hl_liquidations
WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
GROUP BY bucket, token;
```

**Optimized (Using CAGG):**
```sql
-- Uses pre-computed liquidation stats CAGG
SELECT bucket, token, liquidation_count, total_liquidation_value
FROM hl_liquidation_orders_stats_30min
WHERE bucket >= NOW() - INTERVAL '7 days';
-- Pre-aggregated data, no scanning required
```

#### Part 10: Trader Segmentation

**Original (Raw Table):**
```sql
-- Scans 1+ billion rows in hl_user_fills
SELECT address,
       COUNT(*) AS trade_count,
       SUM(size_usd) AS total_volume,
       AVG(size_usd) AS avg_trade_size
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY address;
```

**Optimized (Using CAGG):**
```sql
-- Uses pre-computed traders_30min CAGG
SELECT bucket, address, trade_count, total_volume, avg_trade_size
FROM traders_30min
WHERE bucket >= NOW() - INTERVAL '30 days';
-- Then aggregate by address if needed at query time
```

### MEDIUM Impact Optimizations

These queries would see noticeable improvements but may still need raw data for specific use cases:

#### Part 1: Backtesting - Volume Analysis

```sql
-- Instead of scanning hl_trades for volume:
SELECT bucket, token, total_volume, trade_count, buy_volume, sell_volume
FROM hl_trades_volume_30min
WHERE bucket >= NOW() - INTERVAL '30 days';
```

#### Part 2: Whale Watching - Large Trade Detection

```sql
-- Use traders_30min for activity patterns
-- But still need raw hl_user_fills for individual whale trade detection
SELECT bucket, address, trade_count, total_volume, max_trade_size
FROM traders_30min
WHERE max_trade_size > 100000  -- $100k+ trades
  AND bucket >= NOW() - INTERVAL '7 days';
```

#### Part 7: Time-of-Day Analysis

```sql
-- Use pre-aggregated volume data
SELECT
    EXTRACT(HOUR FROM bucket) AS hour_utc,
    SUM(total_volume) AS volume,
    SUM(trade_count) AS trades
FROM hl_trades_volume_30min
WHERE bucket >= NOW() - INTERVAL '30 days'
GROUP BY hour_utc
ORDER BY volume DESC;
```

### Queries That MUST Use Raw Data

Some analytics require row-level access and cannot use CAGGs:

1. **Part 3: Copy-Trading Detection**
   - Requires precise timestamp correlation between traders
   - Needs individual trade-level data for pattern matching

2. **Part 4: Spread Analysis**
   - Requires tick-by-tick bid/ask data
   - CAGGs aggregate away the granularity needed

3. **Part 5: VaR Calculations**
   - Needs individual returns for percentile calculations
   - CAGGs can provide returns data, but precise VaR needs raw samples

### Available CAGGs Quick Reference

| CAGG Name | Bucket Size | Key Columns |
|-----------|-------------|-------------|
| `funding_rate_1h` | 1 hour | avg_funding_rate, min_rate, max_rate |
| `funding_rate_8h` | 8 hours | avg_funding_rate (standard funding period) |
| `perp_tokens_30min` | 30 min | OHLCV, volume, trade_count |
| `perp_tokens_1h` | 1 hour | OHLCV, volume, trade_count |
| `perp_tokens_1d` | 1 day | OHLCV, volume, trade_count |
| `hl_trades_volume_30min` | 30 min | total_volume, buy/sell_volume, trade_count |
| `traders_30min` | 30 min | address, trade_count, volume, avg_size |
| `hl_liquidation_orders_stats_30min` | 30 min | liq_count, liq_volume, long/short_liqs |
| `perp_asset_positions_v2_1h` | 1 hour | long_oi, short_oi, position_count |
| `hl_c_deposits_stats_30min` | 30 min | deposit_count, total_deposits |
| `hl_c_withdrawals_stats_30min` | 30 min | withdrawal_count, total_withdrawals |

### Performance Comparison Example

```sql
-- SLOW: Raw table query (scans ~1 billion rows)
EXPLAIN ANALYZE
SELECT time_bucket('1 hour', "timestampUTC") AS hour, token,
       COUNT(*), SUM(size_usd)
FROM hl_user_fills
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY hour, token;
-- Execution time: 45-120 seconds

-- FAST: CAGG query (reads ~21,600 pre-computed rows)
EXPLAIN ANALYZE
SELECT bucket, token, trade_count, total_volume
FROM traders_30min
WHERE bucket >= NOW() - INTERVAL '30 days';
-- Execution time: 50-200 milliseconds
```

---

## Recommended New CAGGs to Create

Based on common query patterns in this playbook, here are recommended new Continuous Aggregates that would significantly improve performance:

### 1. Orderbook Spread Stats (HIGH PRIORITY)

**Source Table:** `hl_orderbook`
**Use Case:** Part 4 - Market Microstructure Analysis

```sql
CREATE MATERIALIZED VIEW orderbook_spread_stats_30min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('30 minutes', "timestampUTC") AS bucket,
    token,
    AVG(best_ask - best_bid) AS avg_spread,
    MIN(best_ask - best_bid) AS min_spread,
    MAX(best_ask - best_bid) AS max_spread,
    AVG((best_ask - best_bid) / NULLIF((best_ask + best_bid) / 2, 0) * 100) AS avg_spread_bps,
    AVG(bid_depth_10bps) AS avg_bid_depth,
    AVG(ask_depth_10bps) AS avg_ask_depth,
    COUNT(*) AS snapshot_count
FROM hl_orderbook
GROUP BY bucket, token
WITH NO DATA;

-- Add refresh policy
SELECT add_continuous_aggregate_policy('orderbook_spread_stats_30min',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes');
```

**Expected Speedup:** 50-100x for spread analysis queries

---

### 2. Whale Activity Tracker (HIGH PRIORITY)

**Source Table:** `hl_user_fills`
**Use Case:** Part 2 - Whale Watching, Part 10 - Trader Segmentation

```sql
CREATE MATERIALIZED VIEW whale_activity_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', "timestampUTC") AS bucket,
    address,
    token,
    COUNT(*) AS trade_count,
    SUM(size_usd) AS total_volume,
    MAX(size_usd) AS max_trade_size,
    SUM(CASE WHEN side = 'buy' THEN size_usd ELSE 0 END) AS buy_volume,
    SUM(CASE WHEN side = 'sell' THEN size_usd ELSE 0 END) AS sell_volume,
    AVG(price) AS avg_price
FROM hl_user_fills
WHERE size_usd >= 50000  -- Only track $50k+ trades
GROUP BY bucket, address, token
WITH NO DATA;

-- Add refresh policy
SELECT add_continuous_aggregate_policy('whale_activity_1h',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

**Expected Speedup:** 100x+ for whale detection queries

---

### 3. Hourly Returns for Risk Metrics (MEDIUM PRIORITY)

**Source Table:** `hl_trades` or `perp_tokens`
**Use Case:** Part 5 - Risk Metrics (VaR/CVaR)

```sql
CREATE MATERIALIZED VIEW token_returns_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', "timestampUTC") AS bucket,
    token,
    first(price, "timestampUTC") AS open_price,
    last(price, "timestampUTC") AS close_price,
    (last(price, "timestampUTC") - first(price, "timestampUTC")) /
        NULLIF(first(price, "timestampUTC"), 0) AS hourly_return,
    STDDEV(price) AS price_volatility,
    COUNT(*) AS trade_count
FROM hl_trades
GROUP BY bucket, token
WITH NO DATA;

-- Add refresh policy
SELECT add_continuous_aggregate_policy('token_returns_1h',
    start_offset => INTERVAL '30 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

**Expected Speedup:** 20-50x for VaR calculations

---

### 4. Copy-Trading Correlation Helper (MEDIUM PRIORITY)

**Source Table:** `hl_user_fills`
**Use Case:** Part 3 - Copy-Trading Detection

```sql
CREATE MATERIALIZED VIEW trader_token_activity_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', "timestampUTC") AS bucket,
    address,
    token,
    COUNT(*) AS trade_count,
    SUM(size_usd) AS total_volume,
    STRING_AGG(DISTINCT side, ',') AS sides_traded,
    MIN("timestampUTC") AS first_trade_ts,
    MAX("timestampUTC") AS last_trade_ts
FROM hl_user_fills
GROUP BY bucket, address, token
WITH NO DATA;

-- Add refresh policy (more frequent for real-time detection)
SELECT add_continuous_aggregate_policy('trader_token_activity_5min',
    start_offset => INTERVAL '2 days',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');
```

**Expected Speedup:** 30-50x for correlation detection

---

### 5. Time-of-Day Volume Profile (LOW PRIORITY)

**Source Table:** `hl_trades`
**Use Case:** Part 7 - Time-of-Day/Day-of-Week Analysis

```sql
CREATE MATERIALIZED VIEW volume_by_hour_of_day_1d
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', "timestampUTC") AS bucket,
    token,
    EXTRACT(HOUR FROM "timestampUTC") AS hour_utc,
    EXTRACT(DOW FROM "timestampUTC") AS day_of_week,
    SUM(size_usd) AS volume,
    COUNT(*) AS trade_count,
    AVG(size_usd) AS avg_trade_size
FROM hl_trades
GROUP BY bucket, token, EXTRACT(HOUR FROM "timestampUTC"), EXTRACT(DOW FROM "timestampUTC")
WITH NO DATA;

-- Add refresh policy
SELECT add_continuous_aggregate_policy('volume_by_hour_of_day_1d',
    start_offset => INTERVAL '90 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

**Expected Speedup:** 10-20x for session analysis

---

### 6. Liquidation Cascade Detector (MEDIUM PRIORITY)

**Source Table:** `hl_liquidations`
**Use Case:** Part 8 - Liquidation Cascade Analysis

```sql
CREATE MATERIALIZED VIEW liquidation_intensity_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', "timestampUTC") AS bucket,
    token,
    COUNT(*) AS liq_count,
    SUM(abs_size * price) AS total_liq_value,
    SUM(CASE WHEN side = 'long' THEN abs_size * price ELSE 0 END) AS long_liq_value,
    SUM(CASE WHEN side = 'short' THEN abs_size * price ELSE 0 END) AS short_liq_value,
    AVG(price) AS avg_liq_price,
    MAX(abs_size * price) AS largest_liquidation
FROM hl_liquidations
GROUP BY bucket, token
WITH NO DATA;

-- Add refresh policy (frequent for cascade detection)
SELECT add_continuous_aggregate_policy('liquidation_intensity_5min',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');
```

**Expected Speedup:** 50-100x for cascade analysis

---

### CAGG Creation Priority Summary

| CAGG Name | Source Table | Priority | Est. Speedup | Use Cases |
|-----------|-------------|----------|--------------|-----------|
| `orderbook_spread_stats_30min` | `hl_orderbook` | HIGH | 50-100x | Part 4: Spreads |
| `whale_activity_1h` | `hl_user_fills` | HIGH | 100x+ | Parts 2, 10: Whales |
| `liquidation_intensity_5min` | `hl_liquidations` | MEDIUM | 50-100x | Part 8: Cascades |
| `token_returns_1h` | `hl_trades` | MEDIUM | 20-50x | Part 5: VaR/CVaR |
| `trader_token_activity_5min` | `hl_user_fills` | MEDIUM | 30-50x | Part 3: Copy-Trading |
| `volume_by_hour_of_day_1d` | `hl_trades` | LOW | 10-20x | Part 7: Sessions |

### Storage Considerations

Creating CAGGs trades storage for query speed. Estimated storage impact:

| CAGG | Est. Rows/Day | Est. Storage/Month |
|------|---------------|-------------------|
| `orderbook_spread_stats_30min` | ~1,500 | ~50 MB |
| `whale_activity_1h` | ~10,000 | ~200 MB |
| `liquidation_intensity_5min` | ~8,600 | ~100 MB |
| `token_returns_1h` | ~3,600 | ~75 MB |
| `trader_token_activity_5min` | ~50,000 | ~500 MB |
| `volume_by_hour_of_day_1d` | ~4,000 | ~80 MB |

**Total Estimated Additional Storage:** ~1 GB/month (vs. potential 100x+ query speedups)

---

## Conclusion

This playbook provides a foundation for quantitative analysis of Hyperliquid trading data. Each query can be:
- **Customized**: Adjust time windows, thresholds, and filters
- **Combined**: Build composite signals from multiple indicators
- **Automated**: Create continuous aggregate (CAGG) versions for real-time dashboards

### Key Takeaways

1. **Backtesting is essential** - Always validate strategies with historical data before live trading
2. **Whales provide signals** - Large trader activity often precedes moves
3. **Market microstructure matters** - Spread and flow analysis reveals hidden costs
4. **Risk metrics protect capital** - VaR/CVaR should guide position sizing
5. **Time effects are real** - Session and day-of-week patterns are tradeable
6. **Liquidations cluster** - Cascade risk is highest after initial liquidation waves
7. **Funding extremes revert** - Crowded trades unwind
8. **Different traders, different edges** - Know your trader type and optimize accordingly

---

**Next Steps:**
- Create CAGGs for frequently-used metrics
- Build alerting systems on SQL thresholds
- Backtest combined signals
- Develop risk-adjusted position sizing

*This playbook is part of the Hyperliquid Analytics series. For infrastructure setup, see the companion blog on TimescaleDB Migration.*
