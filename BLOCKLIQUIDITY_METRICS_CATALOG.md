# BlockLiquidity Complete Metrics Catalog

**Comprehensive Guide with Theory, Formulas, SQL Queries & Interpretation**

---

## Table of Contents

1. [Perpetual Markets](#1-perpetual-markets)
2. [Spot Markets](#2-spot-markets)
3. [Trading Activity](#3-trading-activity)
4. [Orderbook & Liquidity](#4-orderbook--liquidity)
5. [Liquidations](#5-liquidations)
6. [Wallet & Portfolio Analytics](#6-wallet--portfolio-analytics)
7. [Risk Metrics](#7-risk-metrics)
8. [Capital Flows](#8-capital-flows)
9. [Leaderboard & Rankings](#9-leaderboard--rankings)
10. [Cross-Exchange Comparison](#10-cross-exchange-comparison)
11. [TWAP & Algorithmic Trading](#11-twap--algorithmic-trading)
12. [Market Sentiment](#12-market-sentiment)
13. [Unrealized PnL Analytics](#13-unrealized-pnl-analytics)
14. [HFT & Market Maker Analytics](#14-hft--market-maker-analytics)
15. [Staking & Delegation](#15-staking--delegation)
16. [Internal Transfers & P2P](#16-internal-transfers--p2p)
17. [Platform Growth Metrics](#17-platform-growth-metrics)
18. [Technical Indicators](#18-technical-indicators)
19. [Backtesting Framework](#19-backtesting-framework)
20. [Copy-Trading Detection](#20-copy-trading-detection)
21. [Time-of-Day Analysis](#21-time-of-day-analysis)

---

## 1. Perpetual Markets

### Source Table: `perp_tokens`

### The Theory

#### What are Perpetual Futures?

Perpetual futures (perps) are derivative contracts that allow traders to speculate on asset prices without expiry dates. Unlike traditional futures that settle on a specific date, perpetuals can be held indefinitely. This is achieved through a mechanism called the **funding rate**.

#### Why Perpetual Markets Matter

1. **Leverage**: Traders can control large positions with small capital (up to 50x on Hyperliquid)
2. **Price Discovery**: Perp prices often lead spot prices due to informed traders
3. **Hedging**: Institutions use perps to hedge spot holdings
4. **Speculation**: Most retail volume is speculative, creating trading opportunities

#### Key Concepts

**Mark Price vs Oracle Price**

| Price Type | Source | Purpose |
|------------|--------|---------|
| **Mark Price** | Exchange's fair value calculation | Used for PnL and liquidations |
| **Oracle Price** | External spot price feeds | Reference for funding calculation |
| **Index Price** | Weighted average from multiple exchanges | Manipulation resistance |

The relationship between mark and oracle prices reveals market sentiment:
- **Mark > Oracle (Premium)**: Bullish sentiment, longs paying shorts
- **Mark < Oracle (Discount)**: Bearish sentiment, shorts paying longs

**Funding Rate Mechanism**

The funding rate is the heart of perpetual futures. It's a periodic payment (every 8 hours on most exchanges) that keeps perp prices anchored to spot.

```
Funding Rate Formula:
─────────────────────
Funding Rate = Premium Index + clamp(Interest Rate - Premium Index, -0.05%, 0.05%)

Where:
• Premium Index = (Perp Price - Spot Price) / Spot Price
• Interest Rate = Base rate (typically 0.01% per 8h)
• clamp() limits extreme values
```

**How Funding Payments Work:**
```
If Funding Rate is POSITIVE:
  → Longs PAY Shorts
  → Encourages shorting to push price down toward spot

If Funding Rate is NEGATIVE:
  → Shorts PAY Longs
  → Encourages longing to push price up toward spot

Payment Amount = Position Size × Funding Rate
```

**Annualizing Funding Rates**

Since funding is paid 3 times per day (every 8 hours):
```
Annualized Rate = 8h Funding Rate × 3 × 365 × 100%

Example:
• 0.01% per 8h = 0.03% per day = ~11% per year
• 0.05% per 8h = 0.15% per day = ~55% per year
• 0.10% per 8h = 0.30% per day = ~110% per year
```

| Funding (8h) | Annualized | Market Interpretation |
|--------------|------------|----------------------|
| > 0.10% | > 110% | Extreme bullish, longs crowded |
| 0.03-0.10% | 33-110% | Bullish sentiment |
| -0.01% to 0.03% | -11% to 33% | Neutral/Normal |
| -0.05% to -0.01% | -55% to -11% | Bearish sentiment |
| < -0.05% | < -55% | Extreme bearish, shorts crowded |

**Open Interest (OI)**

Open Interest represents the total number of outstanding contracts. Unlike volume (which counts both opens and closes), OI only increases when new positions are created.

```
OI Interpretation Matrix:
────────────────────────
| Price Move | OI Change | Interpretation          | Signal Strength |
|------------|-----------|-------------------------|-----------------|
| ↑ Up       | ↑ Up      | New longs entering      | Strong Bullish  |
| ↑ Up       | ↓ Down    | Shorts covering         | Weak Bullish    |
| ↓ Down     | ↑ Up      | New shorts entering     | Strong Bearish  |
| ↓ Down     | ↓ Down    | Longs liquidating       | Weak Bearish    |
```

---

### Direct Metrics from `perp_tokens`

| Column | Metric | Description |
|--------|--------|-------------|
| `token` | Token Symbol | Trading pair (BTC, ETH, etc.) |
| `markPx` | Mark Price | Fair value for PnL calculation |
| `oraclePx` | Oracle Price | External spot reference |
| `funding` | Funding Rate | 8-hour payment rate |
| `dayNtlVlm` | 24h Volume | Daily notional trading volume |
| `openInterest` | Open Interest | Total outstanding contracts |

### SQL Query: Current Perp Market Overview

```sql
SELECT
    token,
    ROUND("markPx"::numeric, 4) AS mark_price,
    ROUND("oraclePx"::numeric, 4) AS oracle_price,
    ROUND((("markPx" - "oraclePx") / NULLIF("oraclePx", 0) * 100)::numeric, 4) AS premium_pct,
    ROUND((funding * 100)::numeric, 4) AS funding_8h_pct,
    ROUND((funding * 3 * 365 * 100)::numeric, 2) AS funding_annualized_pct,
    ROUND("dayNtlVlm"::numeric, 0) AS volume_24h,
    ROUND("openInterest"::numeric, 0) AS open_interest,
    ROUND(("dayNtlVlm" / NULLIF("openInterest", 0))::numeric, 2) AS turnover_ratio
FROM perp_tokens
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
ORDER BY "openInterest" DESC;
```

---

### Derived Metrics

#### 1. Funding Rate Z-Score

**Theory:**
The Z-score measures how many standard deviations the current funding is from its historical mean. Extreme Z-scores (|Z| > 2) indicate unusual market conditions that often precede reversals.

```
Formula:
────────
Z-Score = (Current Value - Mean) / Standard Deviation

Interpretation:
• Z > 2: Extremely high funding, longs very crowded → Consider shorting
• Z > 1: Above average bullish sentiment
• -1 < Z < 1: Normal market conditions
• Z < -1: Below average, bearish sentiment
• Z < -2: Extremely negative funding, shorts crowded → Consider longing
```

**Why It Matters:**
- Extreme funding rates are unsustainable
- Mean reversion is statistically likely
- Provides contrarian trading signals

```sql
WITH funding_stats AS (
    SELECT
        token,
        AVG(funding) AS mean_funding,
        STDDEV(funding) AS std_funding
    FROM perp_tokens
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token
),
current AS (
    SELECT DISTINCT ON (token) token, funding
    FROM perp_tokens
    ORDER BY token, "timestampUTC" DESC
)
SELECT
    c.token,
    ROUND((c.funding * 100)::numeric, 4) AS current_funding_pct,
    ROUND((fs.mean_funding * 100)::numeric, 4) AS avg_funding_pct,
    ROUND(((c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0))::numeric, 2) AS z_score,
    CASE
        WHEN (c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0) > 2 THEN 'EXTREME_BULLISH_CROWDED'
        WHEN (c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0) > 1 THEN 'BULLISH'
        WHEN (c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0) < -2 THEN 'EXTREME_BEARISH_CROWDED'
        WHEN (c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0) < -1 THEN 'BEARISH'
        ELSE 'NEUTRAL'
    END AS signal
FROM current c
JOIN funding_stats fs ON c.token = fs.token
ORDER BY ABS((c.funding - fs.mean_funding) / NULLIF(fs.std_funding, 0)) DESC;
```

---

#### 2. Open Interest Change Analysis

**Theory:**
OI changes combined with price movements reveal who is driving the market. This is fundamental to understanding market structure.

```
The Four Scenarios:
───────────────────

1. PRICE UP + OI UP = "Strong Bullish"
   • New money entering long positions
   • Buyers are aggressive, willing to pay premium
   • Trend likely to continue

2. PRICE UP + OI DOWN = "Short Covering Rally"
   • Shorts are closing losing positions
   • No new buying conviction
   • Rally may be short-lived

3. PRICE DOWN + OI UP = "Strong Bearish"
   • New money entering short positions
   • Sellers are aggressive
   • Downtrend likely to continue

4. PRICE DOWN + OI DOWN = "Long Liquidation"
   • Longs are closing or getting liquidated
   • No new selling conviction
   • May indicate capitulation (potential bottom)
```

```sql
WITH hourly_data AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last("openInterest", "timestampUTC") AS oi,
        last("markPx", "timestampUTC") AS price
    FROM perp_tokens
    WHERE "timestampUTC" >= NOW() - INTERVAL '48 hours'
    GROUP BY token, hour
)
SELECT
    token,
    hour,
    ROUND(oi::numeric, 0) AS open_interest,
    ROUND(price::numeric, 4) AS mark_price,
    ROUND(((oi - LAG(oi) OVER w) / NULLIF(LAG(oi) OVER w, 0) * 100)::numeric, 2) AS oi_change_pct,
    ROUND(((price - LAG(price) OVER w) / NULLIF(LAG(price) OVER w, 0) * 100)::numeric, 2) AS price_change_pct,
    CASE
        WHEN oi > LAG(oi) OVER w AND price > LAG(price) OVER w THEN 'STRONG_BULLISH'
        WHEN oi < LAG(oi) OVER w AND price > LAG(price) OVER w THEN 'SHORT_COVERING'
        WHEN oi > LAG(oi) OVER w AND price < LAG(price) OVER w THEN 'STRONG_BEARISH'
        WHEN oi < LAG(oi) OVER w AND price < LAG(price) OVER w THEN 'LONG_LIQUIDATION'
        ELSE 'NEUTRAL'
    END AS market_regime
FROM hourly_data
WINDOW w AS (PARTITION BY token ORDER BY hour)
ORDER BY token, hour DESC;
```

---

### Source Table: `perp_asset_positions_v2`

### The Theory

#### Understanding Position Data

Position data reveals the actual holdings of traders. This is the "ground truth" of market structure - who is long, who is short, and at what size.

#### Key Position Metrics

**Position Size (szi)**
```
szi > 0 → Long position (profits when price rises)
szi < 0 → Short position (profits when price falls)
szi = 0 → No position
```

**Position Value**
```
Position Value = |Size| × Current Price
This represents the notional exposure in USD
```

**Entry Price**
```
The average price at which the position was opened
Used to calculate unrealized PnL:

Unrealized PnL (Long) = (Current Price - Entry Price) × Size
Unrealized PnL (Short) = (Entry Price - Current Price) × |Size|
```

**Liquidation Price**
```
The price at which the position will be forcefully closed
due to insufficient margin.

For Longs: Liq Price < Entry Price (price drops = liquidation)
For Shorts: Liq Price > Entry Price (price rises = liquidation)

Distance to Liquidation = |Current Price - Liq Price| / Current Price
```

**Leverage**
```
Leverage = Position Value / Margin Used

Example:
• $10,000 position with $1,000 margin = 10x leverage
• Higher leverage = closer liquidation price = higher risk
```

---

### Derived Metrics

#### 1. Long/Short Ratio

**Theory:**
The Long/Short ratio measures the relative positioning between bulls and bears. Extreme readings often precede reversals due to crowding.

```
Formula:
────────
L/S Ratio = Total Long OI / Total Short OI

Interpretation:
• > 2.0: Heavily long-biased, short squeeze unlikely, long squeeze risk HIGH
• 1.2 - 2.0: Moderately long-biased
• 0.8 - 1.2: Balanced market
• 0.5 - 0.8: Moderately short-biased
• < 0.5: Heavily short-biased, short squeeze risk HIGH
```

**Why It Matters:**
- Extreme ratios indicate crowded trades
- Crowded trades are vulnerable to squeezes
- Provides contrarian signal

```sql
SELECT
    token,
    ROUND(SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END)::numeric, 0) AS long_oi,
    ROUND(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)::numeric, 0) AS short_oi,
    ROUND(SUM(ABS(position_value))::numeric, 0) AS total_oi,
    ROUND((SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
           NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0))::numeric, 2) AS long_short_ratio,
    CASE
        WHEN SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
             NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0) > 2 THEN 'EXTREMELY_LONG'
        WHEN SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
             NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0) > 1.5 THEN 'LONG_BIASED'
        WHEN SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
             NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0) < 0.5 THEN 'EXTREMELY_SHORT'
        WHEN SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) /
             NULLIF(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END), 0) < 0.67 THEN 'SHORT_BIASED'
        ELSE 'BALANCED'
    END AS market_bias
FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY token
HAVING SUM(ABS(position_value)) > 1000000
ORDER BY total_oi DESC;
```

---

#### 2. Crowding Score

**Theory:**
The crowding score normalizes the long/short imbalance to a -1 to +1 scale, making it easier to compare across tokens.

```
Formula:
────────
Crowding Score = (Long OI - Short OI) / Total OI

Scale:
• +1.0 = All positions are long (maximum crowding)
• +0.5 = 75% long, 25% short
•  0.0 = Perfectly balanced (50/50)
• -0.5 = 25% long, 75% short
• -1.0 = All positions are short (maximum crowding)

Risk Thresholds:
• > +0.4 = Long squeeze risk (too many longs)
• < -0.4 = Short squeeze risk (too many shorts)
```

**Why It Matters:**
- Normalized metric allows cross-token comparison
- Easy to identify squeeze candidates
- Historical patterns often repeat

```sql
SELECT
    token,
    COUNT(*) AS total_positions,
    ROUND(SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END)::numeric, 0) AS long_oi,
    ROUND(SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)::numeric, 0) AS short_oi,
    ROUND(((SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) -
            SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)) /
           NULLIF(SUM(ABS(position_value)), 0))::numeric, 3) AS crowding_score,
    CASE
        WHEN ((SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) -
               SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)) /
              NULLIF(SUM(ABS(position_value)), 0)) > 0.4 THEN 'LONG_SQUEEZE_RISK'
        WHEN ((SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) -
               SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)) /
              NULLIF(SUM(ABS(position_value)), 0)) < -0.4 THEN 'SHORT_SQUEEZE_RISK'
        ELSE 'BALANCED'
    END AS squeeze_risk
FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY token
HAVING SUM(ABS(position_value)) > 1000000
ORDER BY ABS(((SUM(CASE WHEN szi > 0 THEN position_value ELSE 0 END) -
               SUM(CASE WHEN szi < 0 THEN ABS(position_value) ELSE 0 END)) /
              NULLIF(SUM(ABS(position_value)), 0))) DESC;
```

---

#### 3. Liquidation Risk Analysis

**Theory:**
Distance to liquidation measures how close positions are to being forcefully closed. Clusters of positions with similar liquidation prices create "liquidation walls" that can trigger cascades.

```
Formula:
────────
Distance to Liq % = |Entry Price - Liquidation Price| / Entry Price × 100

Risk Levels:
• < 5%: HIGH RISK - Small price move triggers liquidation
• 5-10%: MODERATE RISK - Vulnerable to volatile moves
• 10-20%: MANAGEABLE RISK - Normal trading conditions
• > 20%: LOW RISK - Well-margined position

Cascade Risk:
When many positions have liquidation prices clustered together,
a price move to that level triggers sequential liquidations,
creating a "cascade" that accelerates the price move.
```

```sql
SELECT
    token,
    CASE WHEN szi > 0 THEN 'LONG' ELSE 'SHORT' END AS direction,
    COUNT(*) AS positions,
    ROUND(SUM(ABS(position_value))::numeric, 0) AS total_value,
    ROUND(AVG(ABS(entry_px - liquidation_px) / NULLIF(entry_px, 0) * 100)::numeric, 2) AS avg_distance_pct,
    COUNT(*) FILTER (WHERE ABS(entry_px - liquidation_px) / NULLIF(entry_px, 0) * 100 < 5) AS high_risk_count,
    COUNT(*) FILTER (WHERE ABS(entry_px - liquidation_px) / NULLIF(entry_px, 0) * 100 BETWEEN 5 AND 10) AS moderate_risk_count,
    ROUND(SUM(ABS(position_value)) FILTER (
        WHERE ABS(entry_px - liquidation_px) / NULLIF(entry_px, 0) * 100 < 10
    )::numeric, 0) AS value_at_risk_10pct
FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
  AND liquidation_px > 0
GROUP BY token, CASE WHEN szi > 0 THEN 'LONG' ELSE 'SHORT' END
ORDER BY value_at_risk_10pct DESC NULLS LAST;
```

---

#### 4. Whale Position Tracking

**Theory:**
Large traders ("whales") often have better information and resources. Tracking their positioning can provide alpha, but it requires distinguishing between different types of large players.

```
Whale Classification:
─────────────────────
| Tier        | Position Size    | Typical Profile                    |
|-------------|------------------|-------------------------------------|
| Mega Whale  | > $1,000,000     | Institutions, market makers         |
| Whale       | $100k - $1M      | Sophisticated traders, funds        |
| Dolphin     | $10k - $100k     | Active traders, semi-professional   |
| Retail      | < $10k           | Individual traders                  |

Why Whales Matter:
• Information Edge: Better research, insider knowledge
• Price Impact: Large orders move markets
• Liquidity Provider: Often act as de facto market makers
• Smart Money: Historically outperform retail
```

```sql
SELECT
    token,
    COUNT(*) FILTER (WHERE ABS(position_value) > 1000000) AS mega_whales,
    COUNT(*) FILTER (WHERE ABS(position_value) BETWEEN 100000 AND 1000000) AS whales,
    COUNT(*) FILTER (WHERE ABS(position_value) BETWEEN 10000 AND 100000) AS dolphins,
    COUNT(*) FILTER (WHERE ABS(position_value) < 10000) AS retail,

    -- Whale net positioning
    ROUND(SUM(position_value) FILTER (WHERE ABS(position_value) > 100000)::numeric, 0) AS whale_net_position,

    -- Whale dominance
    ROUND((SUM(ABS(position_value)) FILTER (WHERE ABS(position_value) > 100000) /
           NULLIF(SUM(ABS(position_value)), 0) * 100)::numeric, 1) AS whale_dominance_pct,

    -- Whale vs Retail sentiment
    CASE
        WHEN SUM(position_value) FILTER (WHERE ABS(position_value) > 100000) > 0
             AND SUM(position_value) FILTER (WHERE ABS(position_value) < 10000) < 0
        THEN 'WHALES_LONG_RETAIL_SHORT'
        WHEN SUM(position_value) FILTER (WHERE ABS(position_value) > 100000) < 0
             AND SUM(position_value) FILTER (WHERE ABS(position_value) < 10000) > 0
        THEN 'WHALES_SHORT_RETAIL_LONG'
        ELSE 'ALIGNED'
    END AS whale_retail_divergence
FROM perp_asset_positions_v2
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY token
ORDER BY whale_dominance_pct DESC;
```

---

## 2. Spot Markets

### Source Table: `spot_tokens`

### The Theory

#### Spot vs Perpetual Markets

Spot markets represent direct ownership of assets, while perpetuals are derivatives. Key differences:

| Aspect | Spot | Perpetual |
|--------|------|-----------|
| Ownership | Own the actual asset | Contract only |
| Leverage | Usually 1x (no leverage) | Up to 50x |
| Funding | None | Pay/receive funding |
| Settlement | Immediate | Continuous |
| Risk | Limited to investment | Can lose more than investment |

#### Why Spot Markets Matter

1. **True Price Discovery**: Spot prices are the "real" asset prices
2. **Reference for Derivatives**: Perp funding is based on spot prices
3. **Lower Manipulation Risk**: Harder to manipulate without leverage
4. **Long-term Holding**: Better for investment vs trading

---

### Direct Metrics

| Column | Metric | Description |
|--------|--------|-------------|
| `token` | Token Symbol | Asset identifier |
| `markPx` | Current Price | Latest trading price |
| `midPx` | Mid Price | (Best Bid + Best Ask) / 2 |
| `prevDayPx` | Previous Day Price | Price 24 hours ago |
| `dayNtlVlm` | 24h Volume | Trading volume in USD |
| `circulatingSupply` | Circulating Supply | Tokens in circulation |
| `marketCap` | Market Cap | Price × Circulating Supply |

---

### Derived Metrics

#### 1. 24-Hour Price Change

**Theory:**
The daily price change is the most fundamental momentum metric. It shows the direction and magnitude of recent price movement.

```
Formula:
────────
24h Change % = (Current Price - Previous Day Price) / Previous Day Price × 100

Interpretation:
• > +10%: Strong rally, potential overbought
• +5% to +10%: Bullish momentum
• -5% to +5%: Consolidation
• -10% to -5%: Bearish momentum
• < -10%: Strong selloff, potential oversold
```

```sql
SELECT
    token,
    ROUND("markPx"::numeric, 6) AS current_price,
    ROUND("prevDayPx"::numeric, 6) AS prev_day_price,
    ROUND((("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100)::numeric, 2) AS change_24h_pct,
    ROUND("dayNtlVlm"::numeric, 0) AS volume_24h,
    ROUND("marketCap"::numeric, 0) AS market_cap,
    CASE
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 > 20 THEN 'PARABOLIC'
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 > 10 THEN 'STRONG_RALLY'
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 > 5 THEN 'RALLY'
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 < -20 THEN 'CRASH'
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 < -10 THEN 'STRONG_SELLOFF'
        WHEN ("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100 < -5 THEN 'SELLOFF'
        ELSE 'STABLE'
    END AS price_action
FROM spot_tokens
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
ORDER BY ABS(("markPx" - "prevDayPx") / NULLIF("prevDayPx", 0) * 100) DESC;
```

---

#### 2. Volume/Market Cap Ratio

**Theory:**
This ratio measures trading intensity relative to market size. High ratios indicate active trading interest.

```
Formula:
────────
Volume/MCap Ratio = 24h Volume / Market Cap

Interpretation:
• > 0.50: Extremely high activity (often during news/events)
• 0.10 - 0.50: High activity
• 0.01 - 0.10: Normal activity
• < 0.01: Low activity (illiquid or stale)

Why It Matters:
• High ratio = High liquidity, easy to enter/exit
• Low ratio = Low liquidity, potential slippage
• Sudden ratio spikes often precede big moves
```

```sql
SELECT
    token,
    ROUND("dayNtlVlm"::numeric, 0) AS volume_24h,
    ROUND("marketCap"::numeric, 0) AS market_cap,
    ROUND(("dayNtlVlm" / NULLIF("marketCap", 0) * 100)::numeric, 2) AS volume_mcap_pct,
    CASE
        WHEN "dayNtlVlm" / NULLIF("marketCap", 0) > 0.5 THEN 'EXTREME_ACTIVITY'
        WHEN "dayNtlVlm" / NULLIF("marketCap", 0) > 0.1 THEN 'HIGH_ACTIVITY'
        WHEN "dayNtlVlm" / NULLIF("marketCap", 0) > 0.01 THEN 'NORMAL'
        ELSE 'LOW_ACTIVITY'
    END AS activity_level
FROM spot_tokens
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
  AND "marketCap" > 0
ORDER BY "dayNtlVlm" / NULLIF("marketCap", 0) DESC;
```

---

## 3. Trading Activity

### Source Table: `hl_trades`

### The Theory

#### Trade Data Fundamentals

Every trade represents a meeting of buyers and sellers. Analyzing trade data reveals:
- **Volume**: Total value exchanged
- **Direction**: Who is the aggressor (buyer or seller)
- **Size Distribution**: Retail vs institutional activity
- **Price Formation**: OHLC candles

#### Key Trading Concepts

**Aggressor vs Passive**
```
In every trade, one party is the "aggressor" (takes liquidity)
and one is "passive" (provides liquidity):

Buy Aggressor: Buyer hits the ask → Price pressure UP
Sell Aggressor: Seller hits the bid → Price pressure DOWN

The "side" field indicates the aggressor's direction.
```

**VWAP (Volume Weighted Average Price)**
```
Formula:
────────
VWAP = Σ(Price × Volume) / Σ(Volume)

Why VWAP Matters:
• Institutional benchmark for execution quality
• Fair price considering all volume
• Price above VWAP = Bullish intraday bias
• Price below VWAP = Bearish intraday bias
```

---

### Derived Metrics

#### 1. OHLCV Candles

**Theory:**
Candlestick data is the foundation of technical analysis. Each candle summarizes price action for a time period.

```
Candle Components:
──────────────────
• Open (O): First trade price in period
• High (H): Highest price in period
• Low (L): Lowest price in period
• Close (C): Last trade price in period
• Volume (V): Total value traded in period

Candle Patterns:
• Bullish: Close > Open (green candle)
• Bearish: Close < Open (red candle)
• Doji: Open ≈ Close (indecision)
• Long wick: Rejection at that price level
```

```sql
SELECT
    token,
    time_bucket('1 hour', "timestampUTC") AS bucket,
    first(px, "timestampUTC") AS open,
    MAX(px) AS high,
    MIN(px) AS low,
    last(px, "timestampUTC") AS close,
    SUM(px * sz) AS volume,
    COUNT(*) AS trades,
    ROUND(((last(px, "timestampUTC") - first(px, "timestampUTC")) /
           NULLIF(first(px, "timestampUTC"), 0) * 100)::numeric, 2) AS change_pct,
    CASE
        WHEN last(px, "timestampUTC") > first(px, "timestampUTC") THEN 'BULLISH'
        WHEN last(px, "timestampUTC") < first(px, "timestampUTC") THEN 'BEARISH'
        ELSE 'DOJI'
    END AS candle_type
FROM hl_trades
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
GROUP BY token, bucket
ORDER BY token, bucket DESC;
```

---

#### 2. Buy/Sell Pressure

**Theory:**
Buy/sell pressure measures which side is more aggressive. This is a leading indicator of price direction.

```
Formula:
────────
Buy Pressure = Buy Volume / Total Volume
Sell Pressure = Sell Volume / Total Volume

Net Pressure = Buy Volume - Sell Volume

Interpretation:
• Buy Pressure > 55%: Buyers dominating
• Buy Pressure > 60%: Strong buying pressure
• Buy Pressure < 45%: Sellers dominating
• Buy Pressure < 40%: Strong selling pressure
```

```sql
SELECT
    token,
    time_bucket('1 hour', "timestampUTC") AS hour,
    ROUND(SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY'))::numeric, 0) AS buy_volume,
    ROUND(SUM(px * sz) FILTER (WHERE side IN ('A', 'SELL'))::numeric, 0) AS sell_volume,
    ROUND(SUM(px * sz)::numeric, 0) AS total_volume,
    ROUND((SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) /
           NULLIF(SUM(px * sz), 0) * 100)::numeric, 1) AS buy_pressure_pct,
    ROUND((SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) -
           SUM(px * sz) FILTER (WHERE side IN ('A', 'SELL')))::numeric, 0) AS net_pressure,
    CASE
        WHEN SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) / NULLIF(SUM(px * sz), 0) > 0.60 THEN 'STRONG_BUY'
        WHEN SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) / NULLIF(SUM(px * sz), 0) > 0.55 THEN 'BUY'
        WHEN SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) / NULLIF(SUM(px * sz), 0) < 0.40 THEN 'STRONG_SELL'
        WHEN SUM(px * sz) FILTER (WHERE side IN ('B', 'BUY')) / NULLIF(SUM(px * sz), 0) < 0.45 THEN 'SELL'
        ELSE 'NEUTRAL'
    END AS pressure_signal
FROM hl_trades
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
GROUP BY token, hour
ORDER BY token, hour DESC;
```

---

#### 3. Large Trade Detection (Whale Activity)

**Theory:**
Large trades often indicate institutional or "smart money" activity. Tracking these provides insight into where big players are positioned.

```
Trade Size Classification:
──────────────────────────
| Category   | Size          | Typical Trader           |
|------------|---------------|--------------------------|
| Mega Whale | > $500,000    | Institutions, funds      |
| Whale      | $100k - $500k | Large traders, prop desks|
| Large      | $50k - $100k  | Active traders           |
| Medium     | $10k - $50k   | Semi-professional        |
| Retail     | < $10k        | Individual traders       |

Why It Matters:
• Large trades often precede price moves
• Whale accumulation = Potential bullish
• Whale distribution = Potential bearish
• Clustering of large trades = Important price levels
```

```sql
SELECT
    token,
    "timestampUTC",
    side,
    ROUND(px::numeric, 4) AS price,
    ROUND(sz::numeric, 4) AS size,
    ROUND((px * sz)::numeric, 0) AS trade_value_usd,
    buyer,
    seller,
    CASE
        WHEN px * sz > 500000 THEN 'MEGA_WHALE'
        WHEN px * sz > 100000 THEN 'WHALE'
        WHEN px * sz > 50000 THEN 'LARGE'
        WHEN px * sz > 10000 THEN 'MEDIUM'
        ELSE 'RETAIL'
    END AS trade_category
FROM hl_trades
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
  AND px * sz > 50000
ORDER BY px * sz DESC
LIMIT 100;
```

---

## 4. Orderbook & Liquidity

### Source Table: `hl_orderbook`

### The Theory

#### What is the Orderbook?

The orderbook is a real-time list of all open buy orders (bids) and sell orders (asks). It represents the supply and demand at every price level.

```
Orderbook Structure:
────────────────────
        ASKS (Sell Orders)
        ──────────────────
        $105.00  |  500 units  ← Best Ask (lowest sell)
        $104.50  |  300 units
        $104.00  |  800 units
        ──────────────────
        SPREAD = $1.00 (Best Ask - Best Bid)
        ──────────────────
        $103.00  |  600 units  ← Best Bid (highest buy)
        $102.50  |  400 units
        $102.00  |  700 units
        ──────────────────
        BIDS (Buy Orders)
```

#### Key Orderbook Concepts

**Bid-Ask Spread**
```
Spread = Best Ask - Best Bid
Spread % = Spread / Mid Price × 100

Tight Spread (< 0.05%): High liquidity, efficient market
Wide Spread (> 0.5%): Low liquidity, higher trading cost
```

**Liquidity Depth**
```
Depth at X% = Total orders within X% of mid price

Example: Depth at 1% for BTC at $100,000
• Bid side: All bids from $99,000 to $100,000
• Ask side: All asks from $100,000 to $101,000
• Total Depth = Bid Depth + Ask Depth
```

**Bid-Ask Imbalance**
```
Formula:
────────
Imbalance = (Bid Liquidity - Ask Liquidity) / (Bid + Ask)

Range: -1 to +1
• +1: All bids, no asks (extreme bullish)
• +0.3 to +1: More bids (bullish)
• -0.3 to +0.3: Balanced
• -1 to -0.3: More asks (bearish)
• -1: All asks, no bids (extreme bearish)

Why It Matters:
• Imbalance often predicts short-term price direction
• Aggressive orders deplete one side, pushing price
```

---

### Derived Metrics

#### 1. Liquidity Analysis

```sql
SELECT
    token,
    time_bucket('15 minutes', "timestampUTC") AS bucket,
    ROUND(AVG(total_bid_usdc_0_5pct + total_ask_usdc_0_5pct)::numeric, 0) AS depth_0_5pct,
    ROUND(AVG(total_bid_usdc_1pct + total_ask_usdc_1pct)::numeric, 0) AS depth_1pct,
    ROUND(AVG(total_bid_usdc_2_5pct + total_ask_usdc_2_5pct)::numeric, 0) AS depth_2_5pct,
    ROUND(AVG(total_bid_usdc_5pct + total_ask_usdc_5pct)::numeric, 0) AS depth_5pct,
    ROUND(AVG(total_bid_usdc_10pct + total_ask_usdc_10pct)::numeric, 0) AS depth_10pct,
    -- Imbalance at 1%
    ROUND(AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
              NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0))::numeric, 3) AS imbalance_1pct
FROM hl_orderbook
WHERE "timestampUTC" >= NOW() - INTERVAL '4 hours'
GROUP BY token, bucket
ORDER BY token, bucket DESC;
```

#### 2. Imbalance Signal

```sql
SELECT
    token,
    time_bucket('5 minutes', "timestampUTC") AS bucket,
    ROUND(AVG(net_usdc_1pct)::numeric, 0) AS net_imbalance_1pct,
    ROUND(AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
              NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0))::numeric, 3) AS imbalance_ratio,
    CASE
        WHEN AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
                 NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0)) > 0.4 THEN 'STRONG_BID_SUPPORT'
        WHEN AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
                 NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0)) > 0.2 THEN 'BID_SUPPORT'
        WHEN AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
                 NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0)) < -0.4 THEN 'STRONG_ASK_PRESSURE'
        WHEN AVG((total_bid_usdc_1pct - total_ask_usdc_1pct) /
                 NULLIF(total_bid_usdc_1pct + total_ask_usdc_1pct, 0)) < -0.2 THEN 'ASK_PRESSURE'
        ELSE 'BALANCED'
    END AS signal
FROM hl_orderbook
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
GROUP BY token, bucket
ORDER BY token, bucket DESC;
```

---

## 5. Liquidations

### Source Table: `hl_liquidations`

### The Theory

#### What is a Liquidation?

A liquidation occurs when a leveraged position's losses exceed its margin. The exchange forcefully closes the position to prevent further losses.

```
Liquidation Process:
────────────────────
1. Trader opens leveraged position
2. Price moves against trader
3. Unrealized loss approaches margin
4. Exchange triggers liquidation
5. Position is market sold/bought
6. This sale/buy further moves price
7. Can trigger more liquidations (CASCADE)
```

#### Liquidation Cascade

```
Cascade Mechanics:
──────────────────
Initial liquidation
       ↓
Price impact from forced sell
       ↓
More positions hit liquidation price
       ↓
More forced selling
       ↓
Price accelerates downward
       ↓
Even more liquidations triggered
       ↓
CASCADE

This can compress hours of normal price movement
into minutes or seconds.
```

#### Liquidation as Contrarian Signal

```
Theory:
───────
High liquidation volume often marks local extremes:
• Mass long liquidations → Potential bottom
• Mass short liquidations → Potential top

Why?
• Leveraged traders are the most aggressive
• Their removal creates buying/selling vacuum
• Price often reverses after forced selling exhausted
```

---

### Derived Metrics

#### 1. Liquidation Summary

```sql
SELECT
    token,
    time_bucket('1 hour', "timestampUTC") AS hour,
    COUNT(*) AS liq_count,
    ROUND(SUM(px * sz)::numeric, 0) AS liq_volume_usd,
    COUNT(*) FILTER (WHERE side IN ('B', 'LONG')) AS long_liqs,
    COUNT(*) FILTER (WHERE side IN ('A', 'SHORT')) AS short_liqs,
    ROUND(AVG(px * sz)::numeric, 0) AS avg_liq_size,
    ROUND(MAX(px * sz)::numeric, 0) AS max_liq_size,
    CASE
        WHEN COUNT(*) FILTER (WHERE side IN ('B', 'LONG')) >
             COUNT(*) FILTER (WHERE side IN ('A', 'SHORT')) * 2 THEN 'LONGS_CRUSHED'
        WHEN COUNT(*) FILTER (WHERE side IN ('A', 'SHORT')) >
             COUNT(*) FILTER (WHERE side IN ('B', 'LONG')) * 2 THEN 'SHORTS_CRUSHED'
        ELSE 'MIXED'
    END AS liq_bias
FROM hl_liquidations
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
GROUP BY token, hour
ORDER BY liq_volume_usd DESC;
```

#### 2. Cascade Detection

```sql
WITH liq_windows AS (
    SELECT
        token,
        "timestampUTC",
        px * sz AS liq_value,
        COUNT(*) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
        ) AS liqs_5min,
        SUM(px * sz) OVER (
            PARTITION BY token
            ORDER BY "timestampUTC"
            RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
        ) AS liq_volume_5min
    FROM hl_liquidations
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
)
SELECT
    token,
    "timestampUTC" AS cascade_time,
    liqs_5min AS liquidations_in_5min,
    ROUND(liq_volume_5min::numeric, 0) AS volume_in_5min_usd,
    CASE
        WHEN liqs_5min >= 20 THEN 'EXTREME_CASCADE'
        WHEN liqs_5min >= 10 THEN 'MAJOR_CASCADE'
        WHEN liqs_5min >= 5 THEN 'CASCADE'
        ELSE 'NORMAL'
    END AS severity
FROM liq_windows
WHERE liqs_5min >= 5
ORDER BY liq_volume_5min DESC
LIMIT 50;
```

---

## 7. Risk Metrics

### Source Tables: `wallet_metrics_v2`, `user_stats_daily`, `maximum_drawdown_daily`

### The Theory

#### Why Risk Metrics Matter

"There are old traders and bold traders, but very few old, bold traders."
Risk management separates sustainable traders from those who eventually blow up.

#### Key Risk Metrics Explained

**1. Sharpe Ratio**
```
Formula:
────────
Sharpe = (Return - Risk-Free Rate) / Standard Deviation

What it measures:
Return per unit of TOTAL volatility (both up and down)

Interpretation:
• < 0: Losing money
• 0 - 0.5: Poor
• 0.5 - 1.0: Acceptable
• 1.0 - 2.0: Good
• 2.0 - 3.0: Excellent
• > 3.0: Exceptional (verify it's real!)

Annualization:
Daily Sharpe × √252 = Annual Sharpe
```

**2. Sortino Ratio**
```
Formula:
────────
Sortino = (Return - Risk-Free Rate) / Downside Deviation

What it measures:
Return per unit of DOWNSIDE volatility only

Why it's better:
• Only penalizes harmful volatility (losses)
• Upside volatility is good!
• Sortino > Sharpe indicates positive skew (desirable)
```

**3. Value at Risk (VaR)**
```
Formula (Historical):
─────────────────────
VaR(95%) = 5th percentile of return distribution

What it means:
"We're 95% confident losses won't exceed this amount"

Example:
VaR(95%) = -5% means:
• 95% of days, you lose less than 5%
• 5% of days, you could lose 5% or more

Limitations:
• Says nothing about losses BEYOND VaR
• Assumes normal distribution (crypto isn't!)
```

**4. Conditional VaR (CVaR / Expected Shortfall)**
```
Formula:
────────
CVaR = Average loss when loss exceeds VaR

What it means:
"When things go bad, HOW bad on average?"

Example:
If VaR(95%) = -5% and CVaR(95%) = -8%:
• On the worst 5% of days, you lose 8% on average

Why it's important:
• Captures "tail risk" that VaR misses
• CVaR is always worse than VaR
• More relevant for crypto's fat tails
```

**5. Maximum Drawdown (MDD)**
```
Formula:
────────
MDD = (Peak Value - Trough Value) / Peak Value × 100

What it measures:
Largest peak-to-trough decline before new high

Why it matters:
• Psychologically devastating metric
• Recovery math is brutal:
  - 10% drawdown needs 11% gain to recover
  - 25% drawdown needs 33% gain to recover
  - 50% drawdown needs 100% gain to recover
  - 75% drawdown needs 300% gain to recover
```

---

### SQL Queries

```sql
-- Complete Risk Dashboard
SELECT
    wm.address,
    ROUND(wm.sharpe_ratio_lifetime::numeric, 2) AS sharpe_lifetime,
    ROUND(wm.sortino_ratio_lifetime::numeric, 2) AS sortino_lifetime,
    ROUND(wm.var_95_lifetime::numeric, 4) AS var_95,
    ROUND(wm.cvar_95_lifetime::numeric, 4) AS cvar_95,
    ROUND(wm.sharpe_ratio_30d::numeric, 2) AS sharpe_30d,
    ROUND(wm.sortino_ratio_30d::numeric, 2) AS sortino_30d,
    -- Tail risk ratio
    ROUND((wm.cvar_95_lifetime / NULLIF(wm.var_95_lifetime, 0))::numeric, 2) AS tail_risk_ratio,
    CASE
        WHEN wm.sharpe_ratio_lifetime > 2 THEN 'EXCELLENT'
        WHEN wm.sharpe_ratio_lifetime > 1 THEN 'GOOD'
        WHEN wm.sharpe_ratio_lifetime > 0.5 THEN 'ACCEPTABLE'
        WHEN wm.sharpe_ratio_lifetime > 0 THEN 'POOR'
        ELSE 'LOSING'
    END AS performance_grade
FROM wallet_metrics_v2 wm
WHERE wm.timestamp >= NOW() - INTERVAL '1 day'
ORDER BY wm.sharpe_ratio_lifetime DESC NULLS LAST
LIMIT 100;
```

---

## 8. Capital Flows

### Source Tables: `hl_deposits`, `hl_withdraws`

### The Theory

#### Why Capital Flows Matter

Capital flows represent REAL money entering or leaving the exchange. Unlike price (which can be manipulated with leverage), flows show actual commitment.

```
Flow Interpretation:
────────────────────
Net Inflows (Deposits > Withdrawals):
• Fresh capital entering
• Bullish signal (money coming to trade/invest)
• Often precedes buying pressure

Net Outflows (Withdrawals > Deposits):
• Capital leaving
• Bearish signal (profit-taking or fear)
• Often precedes selling pressure
```

#### Flow Analysis Framework

| Market Phase | Flow Pattern | Interpretation |
|--------------|--------------|----------------|
| Accumulation | Steady deposits, low withdrawals | Smart money building positions |
| Markup | Accelerating deposits | FOMO kicks in |
| Distribution | Rising withdrawals | Smart money taking profit |
| Markdown | Mass withdrawals | Panic, capitulation |

---

### SQL Queries

```sql
-- Daily Net Flow Analysis
WITH daily_flows AS (
    SELECT
        DATE("timestampUTC") AS day,
        SUM(usdc) AS deposits
    FROM hl_deposits
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY day
),
daily_withdrawals AS (
    SELECT
        DATE("timestampUTC") AS day,
        SUM(usdc) AS withdrawals
    FROM hl_withdraws
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY day
)
SELECT
    COALESCE(d.day, w.day) AS day,
    ROUND(COALESCE(d.deposits, 0)::numeric, 0) AS deposits,
    ROUND(COALESCE(w.withdrawals, 0)::numeric, 0) AS withdrawals,
    ROUND((COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0))::numeric, 0) AS net_flow,
    ROUND((COALESCE(d.deposits, 0) / NULLIF(COALESCE(w.withdrawals, 0), 0))::numeric, 2) AS flow_ratio,
    CASE
        WHEN COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0) > 5000000 THEN 'STRONG_INFLOW'
        WHEN COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0) > 1000000 THEN 'INFLOW'
        WHEN COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0) < -5000000 THEN 'STRONG_OUTFLOW'
        WHEN COALESCE(d.deposits, 0) - COALESCE(w.withdrawals, 0) < -1000000 THEN 'OUTFLOW'
        ELSE 'NEUTRAL'
    END AS flow_signal
FROM daily_flows d
FULL OUTER JOIN daily_withdrawals w ON d.day = w.day
ORDER BY day DESC;
```

---

## 13. Unrealized PnL Analytics

### Source Tables: `hl_unrealized_pnl_per_coin`, `hl_unrealized_pnl_per_tier`

### The Theory

#### What is Unrealized PnL?

Unrealized PnL (Profit and Loss) represents the paper gains or losses on positions that haven't been closed yet. Unlike realized PnL (which is locked in when you close a position), unrealized PnL fluctuates with market prices.

```
Unrealized PnL Calculation:
───────────────────────────
For LONG positions:
  Unrealized PnL = (Current Price - Entry Price) × Position Size

For SHORT positions:
  Unrealized PnL = (Entry Price - Current Price) × Position Size

Example:
• Long 1 BTC at $50,000, current price $55,000
  Unrealized PnL = ($55,000 - $50,000) × 1 = +$5,000 (profit)

• Short 2 ETH at $3,000, current price $3,200
  Unrealized PnL = ($3,000 - $3,200) × 2 = -$400 (loss)
```

#### Why Aggregate Unrealized PnL Matters

The aggregate unrealized PnL across all traders reveals market-wide positioning and sentiment:

```
Market Sentiment from Unrealized PnL:
─────────────────────────────────────
Total Long Unrealized PnL > Total Short Unrealized PnL:
→ Longs are "in the money" (winning)
→ Market has moved up since positions opened
→ Potential resistance: longs may take profit

Total Short Unrealized PnL > Total Long Unrealized PnL:
→ Shorts are "in the money" (winning)
→ Market has moved down since positions opened
→ Potential support: shorts may take profit
```

#### Pain Threshold Analysis

```
Pain Threshold Theory:
─────────────────────
When unrealized losses become large enough, traders are forced
to close positions (voluntarily or via liquidation).

Key Thresholds:
• -10% unrealized: Traders start getting nervous
• -25% unrealized: Many retail traders capitulate
• -50% unrealized: Forced liquidations for leveraged positions
• -75% unrealized: Maximum pain, potential capitulation bottom

The "pain index" measures how much the average trader is underwater.
```

#### Tier-Based Analysis

Different size traders behave differently:

| Tier | Behavior When Underwater | Signal Value |
|------|-------------------------|--------------|
| Small (<$1k) | Quick to panic sell | Contrarian indicator |
| Medium ($1k-$10k) | Moderate conviction | Sentiment gauge |
| Large ($10k-$100k) | Higher pain tolerance | Strong signal |
| Whale (>$100k) | Strategic, informed | Follow closely |

---

### Direct Metrics from `hl_unrealized_pnl_per_coin`

| Column | Metric | Description |
|--------|--------|-------------|
| `coin` | Token | Cryptocurrency being analyzed |
| `total_long` | Total Long PnL | Aggregate unrealized PnL of all long positions |
| `total_short` | Total Short PnL | Aggregate unrealized PnL of all short positions |
| `number_of_trades` | Position Count | Number of open positions |

### Direct Metrics from `hl_unrealized_pnl_per_tier`

| Column | Metric | Description |
|--------|--------|-------------|
| `tier` | Account Tier | Size classification of traders |
| `number_of_address` | Address Count | Number of wallets in this tier |
| `total_long` | Tier Long PnL | Aggregate long PnL for this tier |
| `total_short` | Tier Short PnL | Aggregate short PnL for this tier |

---

### SQL Queries

#### 1. Market-Wide Unrealized PnL by Coin

```sql
SELECT
    coin,
    ROUND(total_long::numeric, 0) AS unrealized_long_pnl,
    ROUND(total_short::numeric, 0) AS unrealized_short_pnl,
    ROUND((total_long + total_short)::numeric, 0) AS net_unrealized_pnl,
    number_of_trades AS open_positions,
    ROUND((total_long / NULLIF(ABS(total_long) + ABS(total_short), 0) * 100)::numeric, 1) AS long_pnl_share_pct,
    CASE
        WHEN total_long > 0 AND total_short < 0 THEN 'LONGS_WINNING'
        WHEN total_long < 0 AND total_short > 0 THEN 'SHORTS_WINNING'
        WHEN total_long > 0 AND total_short > 0 THEN 'ALL_WINNING'
        WHEN total_long < 0 AND total_short < 0 THEN 'ALL_LOSING'
        ELSE 'NEUTRAL'
    END AS market_state,
    CASE
        WHEN total_long > ABS(total_short) * 2 THEN 'STRONG_BULL_REGIME'
        WHEN total_long > ABS(total_short) THEN 'BULL_REGIME'
        WHEN ABS(total_short) > total_long * 2 THEN 'STRONG_BEAR_REGIME'
        WHEN ABS(total_short) > total_long THEN 'BEAR_REGIME'
        ELSE 'NEUTRAL'
    END AS regime
FROM hl_unrealized_pnl_per_coin
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
ORDER BY ABS(total_long + total_short) DESC;
```

#### 2. Tier-Based Sentiment Analysis

```sql
SELECT
    tier,
    number_of_address AS traders,
    ROUND(total_long::numeric, 0) AS tier_long_pnl,
    ROUND(total_short::numeric, 0) AS tier_short_pnl,
    ROUND((total_long + total_short)::numeric, 0) AS tier_net_pnl,
    ROUND(((total_long + total_short) / NULLIF(number_of_address, 0))::numeric, 0) AS avg_pnl_per_trader,
    CASE
        WHEN (total_long + total_short) / NULLIF(number_of_address, 0) > 10000 THEN 'VERY_PROFITABLE'
        WHEN (total_long + total_short) / NULLIF(number_of_address, 0) > 1000 THEN 'PROFITABLE'
        WHEN (total_long + total_short) / NULLIF(number_of_address, 0) > -1000 THEN 'BREAKEVEN'
        WHEN (total_long + total_short) / NULLIF(number_of_address, 0) > -10000 THEN 'UNDERWATER'
        ELSE 'DEEP_UNDERWATER'
    END AS tier_status
FROM hl_unrealized_pnl_per_tier
WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
ORDER BY tier;
```

#### 3. Smart Money vs Retail Divergence

```sql
WITH tier_summary AS (
    SELECT
        CASE
            WHEN tier IN ('tier1', 'small', '0-1k') THEN 'RETAIL'
            WHEN tier IN ('tier5', 'whale', '100k+', 'mega') THEN 'WHALE'
            ELSE 'MIDDLE'
        END AS tier_group,
        SUM(total_long) AS group_long_pnl,
        SUM(total_short) AS group_short_pnl,
        SUM(number_of_address) AS group_traders
    FROM hl_unrealized_pnl_per_tier
    WHERE "timestampUTC" >= NOW() - INTERVAL '1 hour'
    GROUP BY CASE
        WHEN tier IN ('tier1', 'small', '0-1k') THEN 'RETAIL'
        WHEN tier IN ('tier5', 'whale', '100k+', 'mega') THEN 'WHALE'
        ELSE 'MIDDLE'
    END
)
SELECT
    tier_group,
    ROUND(group_long_pnl::numeric, 0) AS long_pnl,
    ROUND(group_short_pnl::numeric, 0) AS short_pnl,
    ROUND((group_long_pnl + group_short_pnl)::numeric, 0) AS net_pnl,
    group_traders,
    CASE
        WHEN group_long_pnl + group_short_pnl > 0 THEN 'PROFITABLE'
        ELSE 'LOSING'
    END AS status
FROM tier_summary
ORDER BY tier_group;
```

---

## 14. HFT & Market Maker Analytics

### Source Table: `hft_count_daily`

### The Theory

#### What is High-Frequency Trading (HFT)?

High-frequency trading uses sophisticated algorithms to execute thousands of trades per second, profiting from tiny price discrepancies. HFT firms are the backbone of market liquidity.

```
HFT Characteristics:
────────────────────
• Trade frequency: 1000+ trades per day
• Holding period: Milliseconds to seconds
• Profit per trade: Fractions of a cent
• Edge: Speed, not prediction
• Volume: 50-70% of all market activity

How HFT Makes Money:
1. Market Making: Profit from bid-ask spread
2. Arbitrage: Price differences across venues
3. Latency Arbitrage: Faster access to information
4. Statistical Arbitrage: Mean reversion patterns
```

#### Why HFT Activity Matters

```
HFT as Market Health Indicator:
───────────────────────────────
High HFT Activity:
• Tighter spreads (cheaper to trade)
• Better price discovery
• More efficient markets
• Lower slippage for large orders

Low HFT Activity:
• Wider spreads
• Poor liquidity
• Higher trading costs
• Potential for manipulation
```

#### Market Maker Behavior Patterns

```
Market Maker Risk Management:
─────────────────────────────
In Calm Markets:
• Quote tight spreads
• Provide deep liquidity
• Hold inventory briefly
• Profit consistently

In Volatile Markets:
• Widen spreads (compensation for risk)
• Reduce liquidity (pull quotes)
• May temporarily exit market
• This WORSENS volatility (liquidity spiral)

Key Insight:
When HFT/MM activity drops sharply, expect increased volatility.
This is a leading indicator of market stress.
```

#### Trade Frequency Distribution

| Category | Trades/Day | Classification | Typical Strategy |
|----------|------------|----------------|------------------|
| HFT | 1000+ | Algorithmic | Market making, arbitrage |
| Active | 100-1000 | Day trader | Technical trading |
| Moderate | 10-100 | Swing trader | Position trading |
| Casual | 1-10 | Retail | Buy and hold with trades |
| Inactive | <1 | Holder | Long-term holding |

---

### SQL Queries

#### 1. Daily HFT Activity Summary

```sql
SELECT
    DATE("timestampUTC") AS day,
    SUM(hft_count) AS total_hft_trades,
    COUNT(DISTINCT address) AS active_hft_addresses,
    ROUND(AVG(hft_count)::numeric, 0) AS avg_trades_per_hft,
    ROUND(MAX(hft_count)::numeric, 0) AS max_trades_single_address,
    -- Day-over-day change
    ROUND(((SUM(hft_count) - LAG(SUM(hft_count)) OVER (ORDER BY DATE("timestampUTC"))) /
           NULLIF(LAG(SUM(hft_count)) OVER (ORDER BY DATE("timestampUTC")), 0) * 100)::numeric, 2) AS dod_change_pct
FROM hft_count_daily
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY DATE("timestampUTC")
ORDER BY day DESC;
```

#### 2. Top HFT Traders Identification

```sql
SELECT
    address,
    SUM(hft_count) AS total_trades,
    COUNT(DISTINCT DATE("timestampUTC")) AS active_days,
    ROUND(AVG(hft_count)::numeric, 0) AS avg_daily_trades,
    ROUND(STDDEV(hft_count)::numeric, 0) AS trade_volatility,
    CASE
        WHEN AVG(hft_count) > 10000 THEN 'MEGA_HFT'
        WHEN AVG(hft_count) > 5000 THEN 'MAJOR_HFT'
        WHEN AVG(hft_count) > 1000 THEN 'HFT'
        WHEN AVG(hft_count) > 100 THEN 'ACTIVE_ALGO'
        ELSE 'SEMI_ACTIVE'
    END AS trader_classification
FROM hft_count_daily
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY address
HAVING SUM(hft_count) > 10000
ORDER BY total_trades DESC
LIMIT 50;
```

#### 3. HFT Activity vs Market Conditions

```sql
WITH daily_hft AS (
    SELECT
        DATE("timestampUTC") AS day,
        SUM(hft_count) AS hft_trades,
        COUNT(DISTINCT address) AS hft_participants
    FROM hft_count_daily
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY DATE("timestampUTC")
)
SELECT
    day,
    hft_trades,
    hft_participants,
    ROUND((hft_trades / NULLIF(hft_participants, 0))::numeric, 0) AS trades_per_participant,
    CASE
        WHEN hft_trades > (SELECT AVG(hft_trades) * 1.5 FROM daily_hft) THEN 'HIGH_ACTIVITY'
        WHEN hft_trades < (SELECT AVG(hft_trades) * 0.5 FROM daily_hft) THEN 'LOW_ACTIVITY'
        ELSE 'NORMAL'
    END AS activity_level,
    -- Rolling 7-day average
    ROUND(AVG(hft_trades) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::numeric, 0) AS rolling_7d_avg
FROM daily_hft
ORDER BY day DESC;
```

---

## 15. Staking & Delegation

### Source Table: `delgated_txns`

### The Theory

#### What is Staking/Delegation?

Staking involves locking up cryptocurrency to support network operations (like validation), earning rewards in return. Delegation allows users to stake without running their own validator.

```
Staking Mechanics:
──────────────────
1. User locks tokens with validator
2. Validator uses combined stake for consensus
3. Rewards distributed proportionally
4. Unbonding period to withdraw (usually 7-21 days)

Key Terms:
• Delegate: Lock tokens with a validator
• Undelegate: Begin withdrawal process
• Redelegate: Move stake between validators
• Slash: Penalty for validator misbehavior
```

#### Why Staking Data Matters

```
Staking as Market Signal:
─────────────────────────
High Staking Rate:
• Reduced circulating supply
• Long-term holder conviction
• Lower sell pressure
• Bullish for price

Increasing Undelegations:
• More tokens entering circulation
• Potential sell pressure incoming
• Could signal distribution phase
• Bearish near-term

Staking Rate Formula:
Rate = Total Staked / Total Supply × 100
```

#### Validator Health Metrics

| Metric | Healthy Range | Warning Signs |
|--------|--------------|---------------|
| Uptime | >99% | <95% uptime |
| Commission | 5-15% | >25% commission |
| Self-stake | >5% | <1% self-stake |
| Delegator Count | >100 | Declining count |

---

### Direct Metrics

| Column | Metric | Description |
|--------|--------|-------------|
| `user` | Delegator Address | Wallet doing the staking |
| `validator` | Validator Address | Validator receiving stake |
| `wei` | Amount | Stake amount in wei (÷10^18 for tokens) |
| `is_undelegate` | Direction | True = withdrawing, False = staking |

---

### SQL Queries

#### 1. Daily Staking Summary

```sql
SELECT
    DATE("timestampUTC") AS day,
    COUNT(*) FILTER (WHERE NOT is_undelegate) AS stake_txns,
    COUNT(*) FILTER (WHERE is_undelegate) AS unstake_txns,
    ROUND(SUM(CASE WHEN NOT is_undelegate THEN wei/1e18 ELSE 0 END)::numeric, 2) AS total_staked,
    ROUND(SUM(CASE WHEN is_undelegate THEN wei/1e18 ELSE 0 END)::numeric, 2) AS total_unstaked,
    ROUND((SUM(CASE WHEN NOT is_undelegate THEN wei/1e18 ELSE 0 END) -
           SUM(CASE WHEN is_undelegate THEN wei/1e18 ELSE 0 END))::numeric, 2) AS net_staking,
    CASE
        WHEN SUM(CASE WHEN NOT is_undelegate THEN wei ELSE 0 END) >
             SUM(CASE WHEN is_undelegate THEN wei ELSE 0 END) * 2 THEN 'STRONG_ACCUMULATION'
        WHEN SUM(CASE WHEN NOT is_undelegate THEN wei ELSE 0 END) >
             SUM(CASE WHEN is_undelegate THEN wei ELSE 0 END) THEN 'ACCUMULATION'
        WHEN SUM(CASE WHEN is_undelegate THEN wei ELSE 0 END) >
             SUM(CASE WHEN NOT is_undelegate THEN wei ELSE 0 END) * 2 THEN 'STRONG_DISTRIBUTION'
        WHEN SUM(CASE WHEN is_undelegate THEN wei ELSE 0 END) >
             SUM(CASE WHEN NOT is_undelegate THEN wei ELSE 0 END) THEN 'DISTRIBUTION'
        ELSE 'NEUTRAL'
    END AS staking_trend
FROM delgated_txns
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY DATE("timestampUTC")
ORDER BY day DESC;
```

#### 2. Top Validators by Delegations

```sql
SELECT
    validator,
    COUNT(*) FILTER (WHERE NOT is_undelegate) AS delegation_count,
    COUNT(*) FILTER (WHERE is_undelegate) AS undelegation_count,
    COUNT(DISTINCT "user") AS unique_delegators,
    ROUND(SUM(CASE WHEN NOT is_undelegate THEN wei/1e18 ELSE -wei/1e18 END)::numeric, 2) AS net_stake,
    ROUND(AVG(CASE WHEN NOT is_undelegate THEN wei/1e18 END)::numeric, 2) AS avg_delegation_size,
    -- Retention metric
    ROUND((COUNT(*) FILTER (WHERE NOT is_undelegate)::float /
           NULLIF(COUNT(*), 0) * 100)::numeric, 1) AS stake_retention_pct
FROM delgated_txns
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY validator
ORDER BY net_stake DESC
LIMIT 20;
```

#### 3. Whale Staker Activity

```sql
SELECT
    "user" AS delegator,
    COUNT(*) AS total_txns,
    COUNT(*) FILTER (WHERE NOT is_undelegate) AS stakes,
    COUNT(*) FILTER (WHERE is_undelegate) AS unstakes,
    ROUND(SUM(wei/1e18)::numeric, 2) AS total_volume,
    ROUND(MAX(wei/1e18)::numeric, 2) AS largest_txn,
    COUNT(DISTINCT validator) AS validators_used,
    CASE
        WHEN SUM(wei/1e18) > 1000000 THEN 'MEGA_WHALE'
        WHEN SUM(wei/1e18) > 100000 THEN 'WHALE'
        WHEN SUM(wei/1e18) > 10000 THEN 'DOLPHIN'
        ELSE 'RETAIL'
    END AS staker_tier
FROM delgated_txns
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY "user"
HAVING SUM(wei/1e18) > 10000
ORDER BY total_volume DESC
LIMIT 50;
```

---

## 16. Internal Transfers & P2P

### Source Tables: `hl_internal_transfers`, `hl_spot_transfers`, `hl_account_class_transfers`

### The Theory

#### Understanding Internal Transfers

Internal transfers move funds between accounts without hitting the open market. This includes:

```
Types of Internal Transfers:
────────────────────────────
1. Sub-account Transfers
   • Same user, different accounts
   • Risk isolation purposes
   • No market impact

2. P2P Transfers
   • Direct wallet-to-wallet
   • OTC deals
   • Payment for services/goods

3. Account Class Transfers
   • Between perp and spot accounts
   • Margin reallocation
   • Position management
```

#### Why Internal Transfers Matter

```
Market Intelligence from Transfers:
───────────────────────────────────
Large P2P Transfers:
• May indicate OTC deals
• Whales moving funds without market impact
• Potential indicator of upcoming activity

Sub-account Activity:
• Traders isolating risk
• Preparing for large positions
• Professional trading setup

Cross-margin Transfers:
• Moving funds to perp = Expecting opportunity
• Moving funds to spot = Taking risk off
```

#### Transfer Pattern Analysis

| Pattern | Interpretation | Signal |
|---------|----------------|--------|
| Spot → Perp | Preparing for leveraged trades | Potentially bullish |
| Perp → Spot | Reducing risk exposure | Potentially bearish |
| Large P2P (same direction) | Coordinated whale activity | Follow the flow |
| Fragmented transfers | Attempting to hide activity | Investigate further |

---

### SQL Queries

#### 1. Internal Transfer Summary

```sql
SELECT
    DATE("timestampUTC") AS day,
    COUNT(*) AS transfer_count,
    COUNT(DISTINCT sender) AS unique_senders,
    COUNT(DISTINCT receiver) AS unique_receivers,
    ROUND(SUM(amount)::numeric, 0) AS total_volume,
    ROUND(AVG(amount)::numeric, 0) AS avg_transfer_size,
    ROUND(MAX(amount)::numeric, 0) AS largest_transfer,
    -- P2P vs Self-transfer detection
    COUNT(*) FILTER (WHERE sender != receiver) AS p2p_transfers,
    COUNT(*) FILTER (WHERE sender = receiver) AS self_transfers
FROM hl_internal_transfers
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY DATE("timestampUTC")
ORDER BY day DESC;
```

#### 2. Spot Transfer Analysis

```sql
SELECT
    token,
    DATE("timestampUTC") AS day,
    COUNT(*) AS transfer_count,
    ROUND(SUM(amount)::numeric, 4) AS total_transferred,
    COUNT(DISTINCT sender) AS senders,
    COUNT(DISTINCT receiver) AS receivers,
    ROUND(AVG(amount)::numeric, 4) AS avg_size,
    CASE
        WHEN SUM(amount) > (
            SELECT AVG(daily_sum) * 2 FROM (
                SELECT SUM(amount) AS daily_sum
                FROM hl_spot_transfers
                WHERE token = st.token
                GROUP BY DATE("timestampUTC")
            ) x
        ) THEN 'ABNORMALLY_HIGH'
        ELSE 'NORMAL'
    END AS activity_level
FROM hl_spot_transfers st
WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
GROUP BY token, DATE("timestampUTC")
ORDER BY total_transferred DESC;
```

#### 3. Account Class Flow Analysis

```sql
SELECT
    DATE("timestampUTC") AS day,
    from_class,
    to_class,
    COUNT(*) AS transfer_count,
    ROUND(SUM(amount)::numeric, 0) AS volume,
    COUNT(DISTINCT address) AS unique_users,
    CASE
        WHEN from_class = 'SPOT' AND to_class = 'PERP' THEN 'RISK_ON'
        WHEN from_class = 'PERP' AND to_class = 'SPOT' THEN 'RISK_OFF'
        ELSE 'REBALANCE'
    END AS flow_type
FROM hl_account_class_transfers
WHERE "timestampUTC" >= NOW() - INTERVAL '7 days'
GROUP BY DATE("timestampUTC"), from_class, to_class
ORDER BY day DESC, volume DESC;
```

#### 4. Whale P2P Detection

```sql
SELECT
    sender,
    receiver,
    "timestampUTC",
    ROUND(amount::numeric, 0) AS transfer_amount,
    CASE
        WHEN amount > 1000000 THEN 'MEGA_WHALE'
        WHEN amount > 100000 THEN 'WHALE'
        WHEN amount > 10000 THEN 'LARGE'
        ELSE 'NORMAL'
    END AS transfer_tier,
    -- Check if sender and receiver are related (same prefix/suffix)
    CASE
        WHEN LEFT(sender, 10) = LEFT(receiver, 10) THEN 'POSSIBLY_RELATED'
        ELSE 'INDEPENDENT'
    END AS relationship_hint
FROM hl_internal_transfers
WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
  AND amount > 10000
  AND sender != receiver
ORDER BY amount DESC
LIMIT 100;
```

---

## 17. Platform Growth Metrics

### Source Tables: `wallets_count`, `trader_profitability`

### The Theory

#### Why Platform Growth Matters

Platform growth metrics reveal the health and adoption trajectory of the exchange:

```
Key Growth Indicators:
──────────────────────
1. Wallet Growth
   • New user adoption rate
   • Network effects indicator
   • Leading indicator for volume

2. Active Wallets
   • Engagement level
   • Stickiness of platform
   • Health of user base

3. Profitability Distribution
   • Are users making money?
   • Sustainable ecosystem?
   • Gambler vs trader ratio
```

#### Growth Phase Analysis

```
Platform Lifecycle:
───────────────────
Early Stage (0-10k wallets):
• Rapid growth percentages
• Volatile metrics
• High churn

Growth Stage (10k-100k wallets):
• Steady new user acquisition
• Improving retention
• Network effects kicking in

Mature Stage (100k+ wallets):
• Slower growth percentages
• High absolute numbers
• Focus on retention

The Hyperliquid Sweet Spot:
When daily new wallets > churned wallets consistently,
the platform has achieved product-market fit.
```

#### Profitability as Health Metric

| Profitable Ratio | Health Status | Interpretation |
|------------------|---------------|----------------|
| > 40% | Healthy | Sustainable ecosystem |
| 30-40% | Normal | Typical distribution |
| 20-30% | Concerning | High churn expected |
| < 20% | Unhealthy | Users leaving soon |

---

### SQL Queries

#### 1. Wallet Growth Trend

```sql
SELECT
    DATE("timestampUTC") AS day,
    number_of_wallets AS total_wallets,
    number_of_wallets - LAG(number_of_wallets) OVER (ORDER BY DATE("timestampUTC")) AS new_wallets,
    ROUND(((number_of_wallets - LAG(number_of_wallets) OVER (ORDER BY DATE("timestampUTC")))::float /
           NULLIF(LAG(number_of_wallets) OVER (ORDER BY DATE("timestampUTC")), 0) * 100)::numeric, 3) AS growth_pct,
    -- 7-day rolling average of new wallets
    ROUND(AVG(number_of_wallets - LAG(number_of_wallets) OVER (ORDER BY DATE("timestampUTC")))
          OVER (ORDER BY DATE("timestampUTC") ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::numeric, 0) AS rolling_7d_new
FROM wallets_count
WHERE "timestampUTC" >= NOW() - INTERVAL '90 days'
ORDER BY day DESC;
```

#### 2. Trader Profitability Analysis

```sql
SELECT
    DATE("timestampUTC") AS day,
    count_profit AS profitable_traders,
    count_not_profit AS unprofitable_traders,
    count_profit + count_not_profit AS total_active_traders,
    ROUND(profit_percentage::numeric, 2) AS profitable_pct,
    CASE
        WHEN profit_percentage > 40 THEN 'EXCELLENT_ECOSYSTEM'
        WHEN profit_percentage > 30 THEN 'HEALTHY'
        WHEN profit_percentage > 20 THEN 'CONCERNING'
        ELSE 'UNHEALTHY'
    END AS ecosystem_health,
    -- Day-over-day change in profitability
    ROUND((profit_percentage - LAG(profit_percentage) OVER (ORDER BY DATE("timestampUTC")))::numeric, 2) AS profitability_change
FROM trader_profitability
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
ORDER BY day DESC;
```

#### 3. Platform Health Dashboard

```sql
WITH wallet_growth AS (
    SELECT
        DATE("timestampUTC") AS day,
        MAX(number_of_wallets) AS wallets
    FROM wallets_count
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY DATE("timestampUTC")
),
profitability AS (
    SELECT
        DATE("timestampUTC") AS day,
        AVG(profit_percentage) AS profit_pct,
        SUM(count_profit + count_not_profit) AS active_traders
    FROM trader_profitability
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY DATE("timestampUTC")
)
SELECT
    w.day,
    w.wallets AS total_wallets,
    w.wallets - LAG(w.wallets) OVER (ORDER BY w.day) AS net_new_wallets,
    ROUND(p.profit_pct::numeric, 1) AS profitability_pct,
    p.active_traders,
    -- Engagement ratio
    ROUND((p.active_traders::float / NULLIF(w.wallets, 0) * 100)::numeric, 2) AS engagement_pct,
    -- Overall health score (simple weighted average)
    ROUND((
        (CASE WHEN w.wallets - LAG(w.wallets) OVER (ORDER BY w.day) > 0 THEN 25 ELSE 0 END) +
        (CASE WHEN p.profit_pct > 30 THEN 25 WHEN p.profit_pct > 20 THEN 15 ELSE 0 END) +
        (CASE WHEN p.active_traders::float / NULLIF(w.wallets, 0) > 0.1 THEN 25 ELSE 15 END) +
        25
    )::numeric, 0) AS health_score
FROM wallet_growth w
LEFT JOIN profitability p ON w.day = p.day
ORDER BY w.day DESC;
```

---

## 18. Technical Indicators

### Source Tables: `hl_trades`, `perp_tokens`, `spot_tokens`

### The Theory

#### What are Technical Indicators?

Technical indicators are mathematical calculations based on price, volume, or open interest data. They help identify trends, momentum, and potential reversal points.

```
Technical Analysis Philosophy:
──────────────────────────────
1. Price discounts everything
   • All information is reflected in price
   • News, fundamentals, psychology = Price

2. Prices move in trends
   • Trends persist until reversed
   • "Trend is your friend"

3. History repeats
   • Market psychology is consistent
   • Patterns recur across timeframes
```

#### Essential Indicator Categories

```
Indicator Types:
────────────────
1. TREND INDICATORS
   • Moving Averages (SMA, EMA)
   • MACD
   • ADX
   → Identify direction

2. MOMENTUM INDICATORS
   • RSI (Relative Strength Index)
   • Stochastic
   • Rate of Change
   → Measure speed/strength

3. VOLATILITY INDICATORS
   • Bollinger Bands
   • ATR (Average True Range)
   • Standard Deviation
   → Measure uncertainty

4. VOLUME INDICATORS
   • OBV (On-Balance Volume)
   • Volume Profile
   • VWAP
   → Confirm price moves
```

#### Key Indicator Formulas

**Simple Moving Average (SMA)**
```
SMA(n) = Sum of last n prices / n

Interpretation:
• Price > SMA: Bullish
• Price < SMA: Bearish
• SMA crossovers: Trend changes
```

**Relative Strength Index (RSI)**
```
RSI = 100 - (100 / (1 + RS))
where RS = Average Gain / Average Loss over n periods

Interpretation:
• RSI > 70: Overbought (potential sell)
• RSI < 30: Oversold (potential buy)
• RSI divergence: Trend weakness
```

**Bollinger Bands**
```
Middle Band = 20-period SMA
Upper Band = SMA + (2 × Standard Deviation)
Lower Band = SMA - (2 × Standard Deviation)

Interpretation:
• Price at Upper Band: Overbought
• Price at Lower Band: Oversold
• Band squeeze: Volatility expansion coming
```

---

### SQL Queries

#### 1. Moving Averages

```sql
WITH hourly_prices AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last("markPx", "timestampUTC") AS close_price
    FROM perp_tokens
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, hour
)
SELECT
    token,
    hour,
    close_price,
    ROUND(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW)::numeric, 4) AS sma_24h,
    ROUND(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 167 PRECEDING AND CURRENT ROW)::numeric, 4) AS sma_7d,
    CASE
        WHEN close_price > AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) THEN 'ABOVE_SMA24'
        ELSE 'BELOW_SMA24'
    END AS trend_signal,
    -- Golden/Death Cross
    CASE
        WHEN AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) >
             AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 167 PRECEDING AND CURRENT ROW)
             AND LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW)) OVER (PARTITION BY token ORDER BY hour) <=
                 LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 167 PRECEDING AND CURRENT ROW)) OVER (PARTITION BY token ORDER BY hour)
        THEN 'GOLDEN_CROSS'
        WHEN AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) <
             AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 167 PRECEDING AND CURRENT ROW)
             AND LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW)) OVER (PARTITION BY token ORDER BY hour) >=
                 LAG(AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 167 PRECEDING AND CURRENT ROW)) OVER (PARTITION BY token ORDER BY hour)
        THEN 'DEATH_CROSS'
        ELSE NULL
    END AS cross_signal
FROM hourly_prices
WHERE hour >= NOW() - INTERVAL '7 days'
ORDER BY token, hour DESC;
```

#### 2. RSI Calculation

```sql
WITH price_changes AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last("markPx", "timestampUTC") AS close_price,
        last("markPx", "timestampUTC") - LAG(last("markPx", "timestampUTC")) OVER (PARTITION BY token ORDER BY time_bucket('1 hour', "timestampUTC")) AS price_change
    FROM perp_tokens
    WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY token, time_bucket('1 hour', "timestampUTC")
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
        AVG(gain) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_gain,
        AVG(loss) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_loss
    FROM gains_losses
)
SELECT
    token,
    hour,
    close_price,
    ROUND((100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001))))::numeric, 2) AS rsi_14,
    CASE
        WHEN (100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001)))) > 70 THEN 'OVERBOUGHT'
        WHEN (100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0.0001)))) < 30 THEN 'OVERSOLD'
        ELSE 'NEUTRAL'
    END AS rsi_signal
FROM rsi_calc
WHERE hour >= NOW() - INTERVAL '7 days'
ORDER BY token, hour DESC;
```

#### 3. Bollinger Bands

```sql
WITH hourly_prices AS (
    SELECT
        token,
        time_bucket('1 hour', "timestampUTC") AS hour,
        last("markPx", "timestampUTC") AS close_price
    FROM perp_tokens
    WHERE "timestampUTC" >= NOW() - INTERVAL '14 days'
    GROUP BY token, hour
),
bb_calc AS (
    SELECT
        token,
        hour,
        close_price,
        AVG(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma_20,
        STDDEV(close_price) OVER (PARTITION BY token ORDER BY hour ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS std_20
    FROM hourly_prices
)
SELECT
    token,
    hour,
    ROUND(close_price::numeric, 4) AS price,
    ROUND(sma_20::numeric, 4) AS middle_band,
    ROUND((sma_20 + 2 * std_20)::numeric, 4) AS upper_band,
    ROUND((sma_20 - 2 * std_20)::numeric, 4) AS lower_band,
    -- Band width (volatility)
    ROUND(((sma_20 + 2 * std_20) - (sma_20 - 2 * std_20)) / sma_20 * 100::numeric, 2) AS band_width_pct,
    -- %B indicator (position within bands)
    ROUND(((close_price - (sma_20 - 2 * std_20)) /
           NULLIF((sma_20 + 2 * std_20) - (sma_20 - 2 * std_20), 0))::numeric, 3) AS pct_b,
    CASE
        WHEN close_price > sma_20 + 2 * std_20 THEN 'ABOVE_UPPER_BAND'
        WHEN close_price < sma_20 - 2 * std_20 THEN 'BELOW_LOWER_BAND'
        WHEN close_price > sma_20 THEN 'UPPER_HALF'
        ELSE 'LOWER_HALF'
    END AS band_position
FROM bb_calc
WHERE hour >= NOW() - INTERVAL '7 days'
ORDER BY token, hour DESC;
```

---

## 19. Backtesting Framework

### Source Tables: `hl_trades`, `perp_tokens`, `hl_user_fills`

### The Theory

#### What is Backtesting?

Backtesting simulates how a trading strategy would have performed using historical data. It's essential for validating strategies before risking real capital.

```
Backtesting Process:
────────────────────
1. Define Strategy Rules
   • Entry conditions
   • Exit conditions
   • Position sizing
   • Risk management

2. Simulate Execution
   • Apply rules to historical data
   • Track hypothetical trades
   • Account for fees and slippage

3. Evaluate Performance
   • Calculate returns
   • Measure risk metrics
   • Compare to benchmarks
```

#### Backtesting Pitfalls

```
Common Mistakes:
────────────────
1. LOOK-AHEAD BIAS
   • Using future information
   • Example: Using close price to make decision that would happen at open

2. SURVIVORSHIP BIAS
   • Only testing on assets that still exist
   • Delisted tokens are ignored

3. OVERFITTING
   • Strategy works perfectly on test data
   • Fails in live trading
   • Solution: Out-of-sample testing

4. IGNORING COSTS
   • Fees eat into profits
   • Slippage on large orders
   • Funding rates for leveraged positions

5. UNREALISTIC FILLS
   • Assuming you get exact price
   • Reality: Orderbook impact
```

#### Walk-Forward Analysis

```
Walk-Forward Testing:
─────────────────────
Instead of one backtest:
1. Train strategy on period 1-100
2. Test on period 101-120
3. Re-train on period 21-120
4. Test on period 121-140
... continue ...

This prevents overfitting by ensuring the strategy
works on truly out-of-sample data.
```

---

### SQL Queries

#### 1. Historical Trade Data for Backtesting

```sql
-- Extract OHLCV data for backtesting
SELECT
    token,
    time_bucket('1 hour', "timestampUTC") AS timestamp,
    first(px, "timestampUTC") AS open,
    MAX(px) AS high,
    MIN(px) AS low,
    last(px, "timestampUTC") AS close,
    SUM(px * sz) AS volume_usd,
    COUNT(*) AS trade_count,
    SUM(CASE WHEN side IN ('B', 'BUY') THEN px * sz ELSE 0 END) AS buy_volume,
    SUM(CASE WHEN side IN ('A', 'SELL') THEN px * sz ELSE 0 END) AS sell_volume
FROM hl_trades
WHERE token = 'BTC'
  AND "timestampUTC" >= NOW() - INTERVAL '90 days'
GROUP BY token, time_bucket('1 hour', "timestampUTC")
ORDER BY timestamp;
```

#### 2. Simple Moving Average Crossover Backtest

```sql
WITH price_data AS (
    SELECT
        token,
        time_bucket('4 hours', "timestampUTC") AS ts,
        last("markPx", "timestampUTC") AS price
    FROM perp_tokens
    WHERE token = 'BTC'
      AND "timestampUTC" >= NOW() - INTERVAL '180 days'
    GROUP BY token, time_bucket('4 hours', "timestampUTC")
),
with_signals AS (
    SELECT
        ts,
        price,
        AVG(price) OVER (ORDER BY ts ROWS BETWEEN 5 PRECEDING AND CURRENT ROW) AS sma_fast,
        AVG(price) OVER (ORDER BY ts ROWS BETWEEN 20 PRECEDING AND CURRENT ROW) AS sma_slow,
        CASE
            WHEN AVG(price) OVER (ORDER BY ts ROWS BETWEEN 5 PRECEDING AND CURRENT ROW) >
                 AVG(price) OVER (ORDER BY ts ROWS BETWEEN 20 PRECEDING AND CURRENT ROW)
            THEN 'LONG'
            ELSE 'FLAT'
        END AS signal
    FROM price_data
),
with_trades AS (
    SELECT
        ts,
        price,
        signal,
        LAG(signal) OVER (ORDER BY ts) AS prev_signal,
        CASE
            WHEN signal = 'LONG' AND LAG(signal) OVER (ORDER BY ts) = 'FLAT' THEN 'BUY'
            WHEN signal = 'FLAT' AND LAG(signal) OVER (ORDER BY ts) = 'LONG' THEN 'SELL'
            ELSE NULL
        END AS trade_action
    FROM with_signals
)
SELECT
    ts,
    price,
    signal,
    trade_action,
    -- Track PnL for each trade
    CASE
        WHEN trade_action = 'BUY' THEN -price
        WHEN trade_action = 'SELL' THEN price
        ELSE 0
    END AS cash_flow
FROM with_trades
WHERE trade_action IS NOT NULL
ORDER BY ts;
```

#### 3. Strategy Performance Metrics

```sql
WITH trades AS (
    -- Your strategy trades here (buy/sell with prices and times)
    SELECT * FROM (VALUES
        ('2024-01-01 00:00:00'::timestamp, 'BUY', 42000),
        ('2024-01-15 00:00:00'::timestamp, 'SELL', 43500),
        ('2024-02-01 00:00:00'::timestamp, 'BUY', 41000),
        ('2024-02-15 00:00:00'::timestamp, 'SELL', 45000)
    ) AS t(ts, action, price)
),
paired_trades AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY ts) AS trade_num,
        ts,
        action,
        price,
        LEAD(price) OVER (ORDER BY ts) AS exit_price,
        LEAD(ts) OVER (ORDER BY ts) AS exit_ts
    FROM trades
),
trade_results AS (
    SELECT
        trade_num / 2 + 1 AS trade_id,
        ts AS entry_time,
        exit_ts AS exit_time,
        price AS entry_price,
        exit_price,
        (exit_price - price) / price * 100 AS return_pct
    FROM paired_trades
    WHERE action = 'BUY' AND exit_price IS NOT NULL
)
SELECT
    COUNT(*) AS total_trades,
    COUNT(*) FILTER (WHERE return_pct > 0) AS winning_trades,
    COUNT(*) FILTER (WHERE return_pct <= 0) AS losing_trades,
    ROUND((COUNT(*) FILTER (WHERE return_pct > 0)::float / COUNT(*) * 100)::numeric, 1) AS win_rate,
    ROUND(AVG(return_pct)::numeric, 2) AS avg_return_pct,
    ROUND(AVG(return_pct) FILTER (WHERE return_pct > 0)::numeric, 2) AS avg_win_pct,
    ROUND(AVG(return_pct) FILTER (WHERE return_pct <= 0)::numeric, 2) AS avg_loss_pct,
    ROUND(SUM(return_pct)::numeric, 2) AS total_return_pct,
    ROUND(MAX(return_pct)::numeric, 2) AS best_trade_pct,
    ROUND(MIN(return_pct)::numeric, 2) AS worst_trade_pct
FROM trade_results;
```

---

## 20. Copy-Trading Detection

### Source Tables: `hl_user_fills`, `hl_trades`, `perp_asset_positions_v2`

### The Theory

#### What is Copy-Trading?

Copy-trading involves automatically replicating another trader's positions. While legitimate as a service, detecting copy patterns reveals:

```
Copy-Trading Indicators:
────────────────────────
1. TEMPORAL CORRELATION
   • Two addresses trading same asset
   • Within seconds of each other
   • Same direction (both buy/sell)

2. SIZE CORRELATION
   • Proportional position sizes
   • Example: Address A trades 10 BTC, Address B trades 1 BTC
   • Consistent ratio over time

3. PORTFOLIO CORRELATION
   • Same tokens held
   • Similar allocation percentages
   • Changes happen simultaneously
```

#### Why Detection Matters

```
Use Cases for Copy Detection:
─────────────────────────────
1. For Users
   • Find profitable traders to follow
   • Verify copy-trading services work
   • Identify potential mentors

2. For Analysis
   • Understand market structure
   • Identify coordinated trading
   • Detect potential manipulation

3. For Compliance
   • Wash trading detection
   • Market manipulation surveillance
   • Account linking
```

#### Correlation Thresholds

| Correlation Level | Meaning | Investigation Priority |
|------------------|---------|----------------------|
| > 0.95 | Near-perfect match | Highly likely related |
| 0.80-0.95 | Strong correlation | Probably related |
| 0.50-0.80 | Moderate correlation | Possible relationship |
| < 0.50 | Weak/no correlation | Likely independent |

---

### SQL Queries

#### 1. Temporal Trade Correlation

```sql
WITH trade_pairs AS (
    SELECT
        t1.token,
        t1.buyer AS address_a,
        t2.buyer AS address_b,
        t1."timestampUTC" AS time_a,
        t2."timestampUTC" AS time_b,
        ABS(EXTRACT(EPOCH FROM (t1."timestampUTC" - t2."timestampUTC"))) AS time_diff_seconds,
        t1.side AS side_a,
        t2.side AS side_b,
        t1.px * t1.sz AS value_a,
        t2.px * t2.sz AS value_b
    FROM hl_trades t1
    JOIN hl_trades t2 ON t1.token = t2.token
        AND t1.buyer < t2.buyer  -- Avoid duplicates
        AND ABS(EXTRACT(EPOCH FROM (t1."timestampUTC" - t2."timestampUTC"))) < 60  -- Within 1 minute
        AND t1.side = t2.side  -- Same direction
    WHERE t1."timestampUTC" >= NOW() - INTERVAL '24 hours'
)
SELECT
    address_a,
    address_b,
    COUNT(*) AS matching_trades,
    ROUND(AVG(time_diff_seconds)::numeric, 2) AS avg_time_diff_sec,
    ROUND(CORR(value_a, value_b)::numeric, 3) AS size_correlation,
    CASE
        WHEN COUNT(*) > 10 AND CORR(value_a, value_b) > 0.8 THEN 'HIGHLY_LIKELY_COPY'
        WHEN COUNT(*) > 5 AND CORR(value_a, value_b) > 0.5 THEN 'POSSIBLE_COPY'
        ELSE 'UNLIKELY'
    END AS copy_likelihood
FROM trade_pairs
GROUP BY address_a, address_b
HAVING COUNT(*) >= 5
ORDER BY matching_trades DESC, size_correlation DESC
LIMIT 50;
```

#### 2. Portfolio Similarity Analysis

```sql
WITH current_positions AS (
    SELECT
        wallet_id,
        token,
        szi AS position_size,
        position_value,
        ABS(position_value) / SUM(ABS(position_value)) OVER (PARTITION BY wallet_id) AS allocation_pct
    FROM perp_asset_positions_v2
    WHERE timestamp >= NOW() - INTERVAL '1 hour'
      AND position_value != 0
),
position_pairs AS (
    SELECT
        p1.wallet_id AS wallet_a,
        p2.wallet_id AS wallet_b,
        COUNT(DISTINCT p1.token) AS common_tokens,
        ROUND(AVG(ABS(p1.allocation_pct - p2.allocation_pct))::numeric, 4) AS avg_allocation_diff,
        -- Check if positions are in same direction
        COUNT(*) FILTER (WHERE SIGN(p1.position_size) = SIGN(p2.position_size)) AS same_direction_count
    FROM current_positions p1
    JOIN current_positions p2 ON p1.token = p2.token
        AND p1.wallet_id < p2.wallet_id
    GROUP BY p1.wallet_id, p2.wallet_id
    HAVING COUNT(DISTINCT p1.token) >= 3
)
SELECT
    wallet_a,
    wallet_b,
    common_tokens,
    avg_allocation_diff,
    same_direction_count,
    ROUND((same_direction_count::float / common_tokens)::numeric, 2) AS direction_agreement_pct,
    CASE
        WHEN avg_allocation_diff < 0.05 AND same_direction_count = common_tokens THEN 'MIRROR_PORTFOLIO'
        WHEN avg_allocation_diff < 0.10 AND same_direction_count::float / common_tokens > 0.8 THEN 'STRONG_SIMILARITY'
        WHEN same_direction_count::float / common_tokens > 0.6 THEN 'MODERATE_SIMILARITY'
        ELSE 'LOW_SIMILARITY'
    END AS similarity_level
FROM position_pairs
ORDER BY avg_allocation_diff ASC, common_tokens DESC
LIMIT 100;
```

#### 3. Leader-Follower Detection

```sql
WITH ordered_fills AS (
    SELECT
        address,
        token,
        side,
        px,
        sz,
        "timestampUTC",
        ROW_NUMBER() OVER (PARTITION BY token, side ORDER BY "timestampUTC") AS trade_order
    FROM hl_user_fills
    WHERE "timestampUTC" >= NOW() - INTERVAL '24 hours'
),
leader_candidates AS (
    SELECT
        o1.address AS leader,
        o2.address AS follower,
        o1.token,
        COUNT(*) AS follow_count,
        AVG(EXTRACT(EPOCH FROM (o2."timestampUTC" - o1."timestampUTC"))) AS avg_lag_seconds
    FROM ordered_fills o1
    JOIN ordered_fills o2 ON o1.token = o2.token
        AND o1.side = o2.side
        AND o1.address != o2.address
        AND o2."timestampUTC" > o1."timestampUTC"
        AND o2."timestampUTC" < o1."timestampUTC" + INTERVAL '5 minutes'
        AND o1.trade_order < o2.trade_order
    GROUP BY o1.address, o2.address, o1.token
    HAVING COUNT(*) >= 3
)
SELECT
    leader,
    follower,
    SUM(follow_count) AS total_follows,
    COUNT(DISTINCT token) AS tokens_followed,
    ROUND(AVG(avg_lag_seconds)::numeric, 1) AS avg_reaction_time_sec,
    CASE
        WHEN AVG(avg_lag_seconds) < 10 THEN 'BOT_COPY'
        WHEN AVG(avg_lag_seconds) < 60 THEN 'ACTIVE_COPY'
        WHEN AVG(avg_lag_seconds) < 300 THEN 'DELAYED_COPY'
        ELSE 'LOOSE_FOLLOW'
    END AS copy_type
FROM leader_candidates
GROUP BY leader, follower
ORDER BY total_follows DESC
LIMIT 50;
```

---

## 21. Time-of-Day Analysis

### Source Tables: `hl_trades`, `perp_tokens`, `hl_user_fills`

### The Theory

#### Why Time Matters in Trading

Markets exhibit distinct behavioral patterns based on time. Understanding these patterns provides a statistical edge.

```
Time-Based Patterns:
────────────────────
1. DAILY PATTERNS
   • Asian session: Lower volatility, range-bound
   • European session: Volatility pickup
   • US session: Highest volatility
   • Overnight: Gaps possible

2. WEEKLY PATTERNS
   • Monday: Often continues Friday's trend
   • Mid-week: Trend confirmation
   • Friday: Position squaring, reduced risk

3. MONTHLY PATTERNS
   • Month-end: Rebalancing flows
   • Options expiry: Increased volatility
   • Futures settlement: Price pinning
```

#### Session Definitions (UTC)

| Session | UTC Time | Characteristics |
|---------|----------|-----------------|
| Asia | 00:00-08:00 | Lower volume, Asia-driven |
| Europe | 08:00-16:00 | Increased activity, overlap |
| US | 16:00-24:00 | Highest volume, US-driven |

#### Hour-of-Day Analysis

```
Statistical Edge by Hour:
─────────────────────────
Studies show:
• Volume peaks at 14:00-15:00 UTC (US market open)
• Volatility lowest 04:00-06:00 UTC
• Large moves often happen at session opens
• Lunch hours show reduced activity

Why This Matters:
• Trade during high-volume hours for better fills
• Expect larger moves during session overlaps
• Adjust position sizing for time of day
```

---

### SQL Queries

#### 1. Hourly Volume Profile

```sql
SELECT
    EXTRACT(HOUR FROM "timestampUTC") AS hour_utc,
    COUNT(*) AS trade_count,
    ROUND(SUM(px * sz)::numeric, 0) AS total_volume_usd,
    ROUND(AVG(px * sz)::numeric, 2) AS avg_trade_size,
    COUNT(DISTINCT DATE("timestampUTC")) AS days_counted,
    ROUND(AVG(SUM(px * sz)) OVER ()::numeric, 0) AS overall_avg_hourly_volume,
    ROUND((SUM(px * sz) / AVG(SUM(px * sz)) OVER () * 100)::numeric, 1) AS relative_volume_pct,
    CASE
        WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 0 AND 7 THEN 'ASIA'
        WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 8 AND 15 THEN 'EUROPE'
        ELSE 'US'
    END AS session
FROM hl_trades
WHERE "timestampUTC" >= NOW() - INTERVAL '30 days'
GROUP BY EXTRACT(HOUR FROM "timestampUTC")
ORDER BY hour_utc;
```

#### 2. Day-of-Week Performance

```sql
WITH daily_returns AS (
    SELECT
        DATE("timestampUTC") AS day,
        EXTRACT(DOW FROM "timestampUTC") AS day_of_week,
        first("markPx", "timestampUTC") AS open_price,
        last("markPx", "timestampUTC") AS close_price,
        (last("markPx", "timestampUTC") - first("markPx", "timestampUTC")) /
            NULLIF(first("markPx", "timestampUTC"), 0) * 100 AS daily_return_pct
    FROM perp_tokens
    WHERE token = 'BTC'
      AND "timestampUTC" >= NOW() - INTERVAL '90 days'
    GROUP BY DATE("timestampUTC"), EXTRACT(DOW FROM "timestampUTC")
)
SELECT
    day_of_week,
    CASE day_of_week
        WHEN 0 THEN 'Sunday'
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
    END AS day_name,
    COUNT(*) AS sample_days,
    ROUND(AVG(daily_return_pct)::numeric, 3) AS avg_return_pct,
    ROUND(STDDEV(daily_return_pct)::numeric, 3) AS return_volatility,
    COUNT(*) FILTER (WHERE daily_return_pct > 0) AS positive_days,
    COUNT(*) FILTER (WHERE daily_return_pct < 0) AS negative_days,
    ROUND((COUNT(*) FILTER (WHERE daily_return_pct > 0)::float / COUNT(*) * 100)::numeric, 1) AS positive_pct
FROM daily_returns
GROUP BY day_of_week
ORDER BY day_of_week;
```

#### 3. Session-Based Volatility Analysis

```sql
WITH session_data AS (
    SELECT
        DATE("timestampUTC") AS day,
        CASE
            WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 0 AND 7 THEN 'ASIA'
            WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 8 AND 15 THEN 'EUROPE'
            ELSE 'US'
        END AS session,
        MIN("markPx") AS session_low,
        MAX("markPx") AS session_high,
        first("markPx", "timestampUTC") AS session_open,
        last("markPx", "timestampUTC") AS session_close
    FROM perp_tokens
    WHERE token = 'BTC'
      AND "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY DATE("timestampUTC"), CASE
        WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 0 AND 7 THEN 'ASIA'
        WHEN EXTRACT(HOUR FROM "timestampUTC") BETWEEN 8 AND 15 THEN 'EUROPE'
        ELSE 'US'
    END
)
SELECT
    session,
    COUNT(*) AS sample_sessions,
    ROUND(AVG((session_high - session_low) / NULLIF(session_open, 0) * 100)::numeric, 3) AS avg_range_pct,
    ROUND(AVG((session_close - session_open) / NULLIF(session_open, 0) * 100)::numeric, 3) AS avg_return_pct,
    ROUND(STDDEV((session_close - session_open) / NULLIF(session_open, 0) * 100)::numeric, 3) AS volatility,
    COUNT(*) FILTER (WHERE session_close > session_open) AS bullish_sessions,
    COUNT(*) FILTER (WHERE session_close < session_open) AS bearish_sessions
FROM session_data
GROUP BY session
ORDER BY CASE session WHEN 'ASIA' THEN 1 WHEN 'EUROPE' THEN 2 ELSE 3 END;
```

#### 4. Optimal Trading Hours

```sql
WITH hourly_stats AS (
    SELECT
        EXTRACT(HOUR FROM "timestampUTC") AS hour_utc,
        SUM(px * sz) AS volume,
        STDDEV(px) AS price_volatility,
        COUNT(*) AS trade_count,
        -- Calculate average spread proxy (price movement per trade)
        AVG(ABS(px - LAG(px) OVER (ORDER BY "timestampUTC"))) AS avg_tick_movement
    FROM hl_trades
    WHERE token = 'BTC'
      AND "timestampUTC" >= NOW() - INTERVAL '30 days'
    GROUP BY EXTRACT(HOUR FROM "timestampUTC")
)
SELECT
    hour_utc,
    ROUND(volume::numeric, 0) AS total_volume,
    trade_count,
    ROUND(price_volatility::numeric, 2) AS volatility,
    -- Efficiency score: high volume, low volatility = better fills
    ROUND((volume / NULLIF(price_volatility, 0) / 1000000)::numeric, 2) AS efficiency_score,
    CASE
        WHEN volume > (SELECT AVG(volume) * 1.5 FROM hourly_stats)
             AND price_volatility < (SELECT AVG(price_volatility) FROM hourly_stats) THEN 'EXCELLENT'
        WHEN volume > (SELECT AVG(volume) FROM hourly_stats) THEN 'GOOD'
        WHEN volume < (SELECT AVG(volume) * 0.5 FROM hourly_stats) THEN 'POOR'
        ELSE 'AVERAGE'
    END AS trading_quality
FROM hourly_stats
ORDER BY efficiency_score DESC;
```

---

## Summary: Complete Metrics Count

| Category | Tables | Direct Metrics | Derived Metrics | Total |
|----------|--------|---------------|-----------------|-------|
| Perp Markets | 2 | 15 | 25 | 40 |
| Spot Markets | 3 | 12 | 15 | 27 |
| Trading Activity | 2 | 20 | 25 | 45 |
| Orderbook | 4 | 35 | 15 | 50 |
| Liquidations | 3 | 15 | 15 | 30 |
| Wallet Analytics | 2 | 25 | 15 | 40 |
| Risk Metrics | 3 | 20 | 10 | 30 |
| Capital Flows | 4 | 10 | 15 | 25 |
| Leaderboard | 1 | 16 | 8 | 24 |
| Cross-Exchange | 2 | 0 | 10 | 10 |
| TWAP | 2 | 8 | 5 | 13 |
| Sentiment | - | 0 | 10 | 10 |
| Unrealized PnL | 2 | 8 | 12 | 20 |
| HFT Analytics | 1 | 4 | 10 | 14 |
| Staking/Delegation | 1 | 4 | 12 | 16 |
| Internal Transfers | 3 | 8 | 15 | 23 |
| Platform Growth | 2 | 6 | 10 | 16 |
| Technical Indicators | - | 0 | 20 | 20 |
| Backtesting | - | 0 | 15 | 15 |
| Copy-Trading | - | 0 | 12 | 12 |
| Time-of-Day | - | 0 | 15 | 15 |
| **TOTAL** | **37** | **206** | **289** | **495+** |

---

*Document generated from database schema analysis with comprehensive theory explanations*
*Last updated: January 2026*
