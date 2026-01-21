# Click Data & Metrics Reference

## Overview

This document catalogs all data fields and metrics used across Click products.

---

## Data Sources

| Source | Type | Description |
|--------|------|-------------|
| Solana Geyser | Raw | Blockchain events via Solana Engine |
| Hyperliquid API | Raw | Perps events via HL Indexer |
| DataStreams | Derived | Computed metrics from raw events |
| ClickHouse | Historical | Storage for all events |
| Postgres | User | User data, orders, settings |

---

## 1. Token Metadata (Static)

| Field | Data Type | Source | Update Frequency |
|-------|-----------|--------|------------------|
| `token_address` | string | On-chain | Static |
| `token_name` | string | Token metadata | Static |
| `token_symbol` | string | Token metadata | Static |
| `token_image_url` | string | Token metadata | Static |
| `total_supply` | decimal | On-chain | Realtime (if mints/burns) |
| `decimals` | integer | On-chain | Static |
| `deployment_time` | timestamp | On-chain | Static |
| `creator_wallet` | string | On-chain | Static |
| `website_url` | string | Token metadata | Static |
| `twitter_url` | string | Token metadata | Static |
| `telegram_url` | string | Token metadata | Static |
| `tiktok_url` | string | Token metadata | Static |
| `dex_source` | string | Click Indexer | Static |
| `launchpad` | string | Click Indexer | Static |
| `mint_authority` | string/null | On-chain | Static |
| `freeze_authority` | string/null | On-chain | Static |

---

## 2. Price & Market Data (Realtime)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `current_price_usd` | decimal | Latest trade price | Realtime |
| `price_sol` | decimal | Latest trade price in SOL | Realtime |
| `market_cap_usd` | decimal | `price_usd × total_supply` | Realtime |
| `ath_market_cap` | decimal | `MAX(market_cap_usd)` historical | Realtime |
| `ath_price` | decimal | `MAX(price_usd)` historical | Realtime |
| `price_change_5m_pct` | decimal | `(current - price_5m_ago) / price_5m_ago × 100` | Realtime |
| `price_change_1h_pct` | decimal | Same formula, 1h window | Realtime |
| `price_change_6h_pct` | decimal | Same formula, 6h window | Realtime |
| `price_change_24h_pct` | decimal | Same formula, 24h window | Realtime |

---

## 3. Volume & Liquidity (Realtime)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `volume_24h_usd` | decimal | `SUM(trade_value_usd)` last 24h | Realtime |
| `volume_24h_sol` | decimal | `SUM(trade_volume_sol)` last 24h | Realtime |
| `volume_1h_usd` | decimal | `SUM(trade_value_usd)` last 1h | Realtime |
| `buy_volume_usd` | decimal | `SUM(trade_value_usd)` WHERE type = buy | Realtime |
| `sell_volume_usd` | decimal | `SUM(trade_value_usd)` WHERE type = sell | Realtime |
| `net_buy_volume_usd` | decimal | `buy_volume - sell_volume` | Realtime |
| `buy_count` | integer | `COUNT(*)` WHERE type = buy | Realtime |
| `sell_count` | integer | `COUNT(*)` WHERE type = sell | Realtime |
| `transaction_count_24h` | integer | `COUNT(*)` last 24h | Realtime |
| `trader_count_24h` | integer | `COUNT(DISTINCT wallet)` last 24h | Realtime |
| `liquidity_usd` | decimal | `SUM(pool_reserves_usd)` | Realtime |
| `liquidity_sol` | decimal | Pool reserves in SOL | Realtime |
| `global_fees_paid_usd` | decimal | `SUM(fee_usd)` all time | Realtime |

---

## 4. OHLCV Candle Data (Realtime)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `token_address` | string | From event | Static |
| `timeframe` | enum | Configuration | Static |
| `ohlcv_open` | decimal | First trade price in period | Realtime |
| `ohlcv_high` | decimal | `MAX(price)` in period | Realtime |
| `ohlcv_low` | decimal | `MIN(price)` in period | Realtime |
| `ohlcv_close` | decimal | Last trade price in period | Realtime |
| `ohlcv_volume` | decimal | `SUM(trade_value_usd)` in period | Realtime |
| `candle_timestamp` | timestamp | Period start time | Static |

**Supported Timeframes (14)**:
1s, 5s, 15s, 30s, 1m, 5m, 15m, 30m, 1h, 4h, 6h, 12h, 1d, 1w

---

## 5. Holder Analytics (Realtime)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `holder_count` | integer | `COUNT(DISTINCT wallet)` WHERE balance > 0 | Realtime |
| `top_10_holder_pct` | decimal | `SUM(top10_balance) / supply × 100` | Realtime |
| `top_20_holder_pct` | decimal | `SUM(top20_balance) / supply × 100` | Realtime |
| `dev_holdings_pct` | decimal | `dev_balance / supply × 100` | Realtime |
| `dev_sold` | boolean | `dev_balance == 0 AND has_sold` | Realtime |
| `sniper_pct` | decimal | `SUM(sniper_balance) / supply × 100` | Realtime |
| `insider_pct` | decimal | `SUM(insider_balance) / supply × 100` | Realtime |
| `insider_count` | integer | `COUNT(DISTINCT wallet)` WHERE insider = true | Realtime |
| `bundler_pct` | decimal | `SUM(bundler_balance) / supply × 100` | Realtime |
| `phishing_pct` | decimal | `SUM(phishing_balance) / supply × 100` | Realtime |
| `blacklist_pct` | decimal | `SUM(blacklist_balance) / supply × 100` | Realtime |
| `pro_trader_count` | integer | `COUNT(DISTINCT wallet)` WHERE is_bot = true | Realtime |

---

## 6. Risk Metrics (Realtime)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `rug_risk_pct` | decimal | Weighted: dev_holdings + insider + liquidity | Realtime |
| `burned_liquidity_pct` | decimal | `burned_lp / total_lp × 100` | Realtime |
| `no_mint` | boolean | Mint authority == null | Static |
| `dex_paid` | boolean | DexScreener verification | Periodic (daily) |

---

## 7. Wallet Classification Flags

| Field | Data Type | Definition | Update Frequency |
|-------|-----------|------------|------------------|
| `is_sniper` | boolean | `buy_slot == creation_slot` | Static |
| `is_insider` | boolean | Received tokens via transfer (no buy tx) | Static |
| `is_bundler` | boolean | Coordinated wallet group (same-block, funded together) | Periodic (daily) |
| `is_fresh_wallet` | boolean | `wallet_age < 7d AND tx_count < 10` | Periodic (daily) |
| `is_pro_trader` | boolean | Bot detection algorithm | Periodic (daily) |
| `is_dev` | boolean | Deployed token or received initial supply | Static |
| `is_kol` | boolean | Key Opinion Leader (curated list) | Static |
| `is_whale` | boolean | Large balance threshold | Realtime |
| `is_top_10_holder` | boolean | `holder_rank <= 10` | Realtime |

---

## 8. Developer Information

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `dev_wallet_address` | string | Deployer wallet | Static |
| `funding_source` | string | Exchange or wallet (1 hop trace) | Static |
| `funding_amount_sol` | decimal | SOL amount used to fund | Static |
| `funding_age` | timestamp | When dev wallet was funded | Static |
| `funding_tx_hash` | string | Transaction signature | Static |

---

## 9. Trade Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `tx_signature` | string | Transaction signature | Static |
| `token_address` | string | Token mint | Static |
| `trade_type` | enum | Buy or Sell | Realtime |
| `trade_amount` | decimal | Token amount traded | Realtime |
| `trade_total_usd` | decimal | `amount × price_at_tx` | Realtime |
| `trade_total_sol` | decimal | SOL value | Realtime |
| `trade_mcap_usd` | decimal | `price_at_tx × total_supply` | Realtime |
| `trader_wallet` | string | Wallet address | Static |
| `tx_timestamp` | timestamp | On-chain time | Static |
| `dex_source` | string | Which DEX/pool | Static |

---

## 10. Holder Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `holder_rank` | integer | `RANK() OVER (ORDER BY balance DESC)` | Realtime |
| `holder_wallet` | string | Wallet address | Static |
| `supply_held_pct` | decimal | `balance / total_supply × 100` | Realtime |
| `current_value_usd` | decimal | `balance × current_price_usd` | Realtime |
| `average_entry_usd` | decimal | `SUM(buy_value) / SUM(buy_amount)` | Realtime |
| `realized_pnl_usd` | decimal | `SUM(sell_value - cost_basis)` | Realtime |
| `total_pnl_usd` | decimal | `realized_pnl + unrealized_pnl` | Realtime |
| `remaining_pct` | decimal | `current_balance / peak_balance × 100` | Realtime |
| `funding_address` | string | 1 hop funding source | Static |
| `funding_amount_sol` | decimal | Amount received | Static |

---

## 11. Pool / Liquidity Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `pool_address` | string | Pool address | Static |
| `token_address` | string | Token mint | Static |
| `base_token` | string | SOL/USDC | Static |
| `reserve_token` | decimal | Token reserves | Realtime |
| `reserve_base` | decimal | Base reserves | Realtime |
| `lp_supply` | decimal | Total LP tokens | Realtime |
| `lp_burned` | decimal | Burned LP tokens | Realtime |
| `pool_creation_time` | timestamp | Pool created | Static |
| `amm_type` | string | Raydium/Orca/etc | Static |

---

## 12. Bonding Curve Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `bonding_curve_pct` | decimal | Progress percentage | Realtime |
| `is_graduated` | boolean | `bonding_curve_pct == 100` | Realtime |
| `graduation_time` | timestamp | When graduated | Static |
| `launchpad` | string | Pump.fun, Bonk, etc. | Static |

---

## 13. User Position Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `user_id` | string | Click user ID | Static |
| `token_address` | string | Token | Static |
| `bought_sol` | decimal | `SUM(buy_amount_sol)` | Realtime |
| `sold_sol` | decimal | `SUM(sell_amount_sol)` | Realtime |
| `holding_value_sol` | decimal | `token_balance × current_price_sol` | Realtime |
| `pnl_sol` | decimal | `sold + holding - bought` | Realtime |
| `pnl_pct` | decimal | `pnl / bought × 100` | Realtime |
| `first_buy_timestamp` | timestamp | First purchase | Static |
| `is_hidden` | boolean | User setting | User action |

---

## 14. Perps Position Data (Hyperliquid)

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `position_id` | string | Unique ID | Static |
| `coin` | string | Asset (BTC, ETH, etc.) | Static |
| `direction` | enum | Long / Short | Static |
| `size` | decimal | Position size | Realtime |
| `leverage` | decimal | `position_value / margin` | Realtime |
| `entry_price` | decimal | Volume-weighted average | Static |
| `mark_price` | decimal | Current market price | Realtime |
| `unrealized_pnl_usd` | decimal | `(mark - entry) × size × direction` | Realtime |
| `roe_pct` | decimal | `unrealized_pnl / margin × 100` | Realtime |
| `liq_price` | decimal | Liquidation threshold | Realtime |
| `margin` | decimal | Allocated collateral | Realtime |
| `funding` | decimal | Accumulated funding | Periodic (8h) |

---

## 15. Wallet Data

| Field | Data Type | Calculation | Update Frequency |
|-------|-----------|-------------|------------------|
| `wallet_id` | string | UUID | Static |
| `wallet_address` | string | Solana public key | Static |
| `wallet_number` | integer | Display number (0 = Main) | Static |
| `is_main` | boolean | `wallet_number == 0` | Static |
| `is_hidden` | boolean | User setting | User action |
| `created_at` | timestamp | Creation time | Static |
| `sol_balance` | decimal | SOL balance | Realtime |
| `token_balances` | array | All token holdings | Realtime |
| `total_value_usd` | decimal | `SUM(balances × prices)` | Realtime |

---

## 16. Discovery Section Qualifications

| Section | Qualification Logic |
|---------|---------------------|
| Newly Created | `bonding_curve_pct < 60` |
| About to Graduate | `mcap >= 40000 AND mcap <= 60000 AND bonding_curve_pct >= 60` |
| Graduated | `bonding_curve_pct == 100 AND has_pool` |

---

## Data Update Frequencies

| Frequency | Description | Examples |
|-----------|-------------|----------|
| Static | Never changes after creation | Token address, deployment time |
| Realtime | Updates per event (trade/transfer) | Price, volume, holder count |
| Periodic (daily) | Batch computed daily | Bundler detection |
| Periodic (8h) | Fixed intervals | Funding payments |
| User action | On user input | Hidden positions, settings |

---

## Data Storage Locations

| Storage | Data Types |
|---------|------------|
| ClickHouse | Trades, transfers, tokens, pools, metrics |
| Postgres | Users, orders, settings, wallet associations |
| Redis | Sessions, rate limits, cache |
| DataStreams (memory) | Active token state, computation |
