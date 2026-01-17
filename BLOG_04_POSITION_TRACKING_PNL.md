# Building Real-Time Solana DEX Analytics with ClickHouse: Part 4

## Position Tracking, FIFO PnL Calculations & ReplacingMergeTree Deep Dive

*Part 4 of a comprehensive series on building production-grade DEX analytics infrastructure*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Data Pipeline: Position Tracking Flow](#data-pipeline-position-tracking-flow)
3. [Position Tracking Theory](#position-tracking-theory)
4. [The PositionOverview Data Model](#the-positionoverview-data-model)
5. [Understanding Cost Basis Methods](#understanding-cost-basis-methods)
6. [Realized vs Unrealized PnL](#realized-vs-unrealized-pnl)
7. [Fee Tracking Architecture](#fee-tracking-architecture)
8. [ReplacingMergeTree Deep Dive](#replacingmergetree-deep-dive)
9. [Current State vs Historical Patterns](#current-state-vs-historical-patterns)
10. [Computed Fields Strategy](#computed-fields-strategy)
11. [View Architecture for PnL](#view-architecture-for-pnl)
12. [Query Optimization Patterns](#query-optimization-patterns)
13. [Summary](#summary)

---

## Introduction

Position tracking is arguably the most critical component of any trading analytics platform. While OHLCV candles show market movement and token metrics show aggregate activity, position data answers the questions traders actually care about:

- **"How much am I up/down?"** - PnL calculations
- **"What's my average buy price?"** - Cost basis tracking
- **"What percentage of the supply do I hold?"** - Whale detection
- **"How much have I paid in fees?"** - Total cost analysis

In this article, we'll explore the complete position tracking architecture, including:

- Multi-source position aggregation (trades + transfers)
- FIFO-based cost basis and PnL calculations
- ReplacingMergeTree for state management
- Fee decomposition and accumulation
- View architecture for computed fields

---

## Data Pipeline: Position Tracking Flow

### How Positions Are Generated in DataStreams

Position updates come from TWO sources: trades and transfers:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    POSITION TRACKING PIPELINE                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  INPUT SOURCES:                                                                  │
│  ├── solana.trades.v3      (Swap events)                                        │
│  └── solana.transfers.v2   (Token transfers)                                    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                    TRANSFORM: position_overview_from_trades                  ││
│  │                                                                              ││
│  │  Pipeline: click_users                                                       ││
│  │  Source: solana.trades.v3                                                    ││
│  │  Partition By: trader (parallel processing per wallet)                       ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 1: Load Position State                                            │  ││
│  │  │ ├─ Key: (trader, token_mint)                                          │  ││
│  │  │ ├─ context.state.get::<PositionState>(key)                            │  ││
│  │  │ └─ Returns existing position or creates new                            │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 2: Update Position Based on Trade Direction                       │  ││
│  │  │                                                                        │  ││
│  │  │ If trade.direction == BUY:                                            │  ││
│  │  │ ├─ position.total_tokens_bought += trade.token_amount                 │  ││
│  │  │ ├─ position.total_spent_sol += trade.volume_sol                       │  ││
│  │  │ ├─ position.total_spent_usd += trade.volume_usd                       │  ││
│  │  │ └─ If first buy: position.initial_tokens_bought = trade.token_amount  │  ││
│  │  │                                                                        │  ││
│  │  │ If trade.direction == SELL:                                           │  ││
│  │  │ ├─ position.total_tokens_sold += trade.token_amount                   │  ││
│  │  │ ├─ position.total_received_sol += trade.volume_sol                    │  ││
│  │  │ ├─ position.total_received_usd += trade.volume_usd                    │  ││
│  │  │ └─ Recalculate realized_pnl (see PnL section)                         │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 3: Accumulate Fees                                                │  ││
│  │  │ ├─ position.total_transaction_fees_sol += trade.transaction_fee_sol   │  ││
│  │  │ ├─ position.total_priority_fees_sol += trade.priority_fee_sol         │  ││
│  │  │ ├─ position.total_validator_tips_sol += trade.validator_tip_sol       │  ││
│  │  │ ├─ position.total_dex_fees_sol += trade.dex_fee_sol                   │  ││
│  │  │ ├─ position.total_platform_fees_sol += trade.platform_fee_sol         │  ││
│  │  │ └─ position.total_fees_paid_sol = sum of all above                    │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 4: Recalculate Holdings                                           │  ││
│  │  │ holdings = bought - sold + received - sent                             │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 5: Update Supply Percentage                                       │  ││
│  │  │ ├─ Lookup token supply from state                                     │  ││
│  │  │ └─ position.supply_pct = (holdings / supply) × 100                    │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  │  ┌───────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STEP 6: Save State and Emit                                            │  ││
│  │  │ ├─ context.state.set((trader, token_mint), position)                  │  ││
│  │  │ └─ context.emit("solana.position_overview", position_overview)        │  ││
│  │  └───────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                              ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                    TRANSFORM: position_overview_from_transfers               ││
│  │                                                                              ││
│  │  Pipeline: click_users                                                       ││
│  │  Source: solana.transfers.v2                                                 ││
│  │  Partition By: sender OR receiver (two separate transforms)                  ││
│  │                                                                              ││
│  │  SEND Transform (position_overview_from_send):                               ││
│  │  ├─ Key: (transfer.sender, transfer.token_mint)                             ││
│  │  ├─ position.total_tokens_sent += transfer.amount                           ││
│  │  ├─ Recalculate holdings                                                    ││
│  │  └─ Emit updated position                                                   ││
│  │                                                                              ││
│  │  RECEIVE Transform (position_overview_from_receive):                         ││
│  │  ├─ Key: (transfer.receiver, transfer.token_mint)                           ││
│  │  ├─ position.total_tokens_received += transfer.amount                       ││
│  │  ├─ Recalculate holdings                                                    ││
│  │  └─ Emit updated position                                                   ││
│  │                                                                              ││
│  │  NOTE: Transfers do NOT affect cost basis or PnL!                           ││
│  │  Tokens maintain their original cost basis when transferred.                 ││
│  │                                                                              ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│  OUTPUT: solana.position_overview (28 fields)                                   │
│  Partition Key: trader (for efficient ClickHouse queries)                       │
│  Serializer: RowBinary                                                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### State Management for Positions

Position state is keyed by (trader, token_mint) tuple:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    POSITION STATE STRUCTURE                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  #[derive(AnyState, Serialize, Deserialize)]                                    │
│  pub struct PositionState {                                                     │
│                                                                                  │
│    // ═══════════════════════════════════════════════════════════════════════  │
│    // QUANTITY TRACKING                                                          │
│    // ═══════════════════════════════════════════════════════════════════════  │
│                                                                                  │
│    total_tokens_bought: Decimal,     // Sum of all buy amounts                 │
│    total_tokens_sold: Decimal,       // Sum of all sell amounts                │
│    total_tokens_received: Decimal,   // Sum of incoming transfers              │
│    total_tokens_sent: Decimal,       // Sum of outgoing transfers              │
│    initial_tokens_bought: Decimal,   // First buy amount (immutable)           │
│                                                                                  │
│    // ═══════════════════════════════════════════════════════════════════════  │
│    // VALUE TRACKING (SOL + USD)                                                │
│    // ═══════════════════════════════════════════════════════════════════════  │
│                                                                                  │
│    total_spent_sol: Decimal,         // Total SOL spent on buys                │
│    total_spent_usd: Decimal,         // Total USD spent on buys                │
│    total_received_sol: Decimal,      // Total SOL received from sells          │
│    total_received_usd: Decimal,      // Total USD received from sells          │
│                                                                                  │
│    // ═══════════════════════════════════════════════════════════════════════  │
│    // FEE ACCUMULATION                                                          │
│    // ═══════════════════════════════════════════════════════════════════════  │
│                                                                                  │
│    total_transaction_fees_sol: Decimal,                                         │
│    total_priority_fees_sol: Decimal,                                            │
│    total_validator_tips_sol: Decimal,                                           │
│    total_dex_fees_sol: Decimal,                                                 │
│    total_platform_fees_sol: Decimal,                                            │
│                                                                                  │
│  }                                                                              │
│                                                                                  │
│  State Key: format!("{}:{}", trader, token_mint)                                │
│  Example: "7xKXtg2CED456...:DezXAZ8z7Pnr..."                                    │
│                                                                                  │
│  Memory per position: ~500 bytes                                                │
│  Total for 1M active positions: ~500 MB                                         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse Ingestion for Positions

```sql
-- NATS Engine subscribes to position updates
CREATE TABLE datastreams.position_overview_queue (
    trader String,
    click_user_id Nullable(String),
    token_mint String,
    token_name String,
    token_symbol String,
    holdings_token Decimal(38, 18),
    total_tokens_sent Decimal(38, 18),
    total_tokens_received Decimal(38, 18),
    total_tokens_bought Decimal(38, 18),
    total_tokens_sold Decimal(38, 18),
    initial_tokens_bought Decimal(38, 18),
    total_spent_sol Decimal(38, 18),
    total_spent_usd Decimal(38, 18),
    total_received_sol Decimal(38, 18),
    total_received_usd Decimal(38, 18),
    realized_pnl_sol Decimal(38, 18),
    realized_pnl_usd Decimal(38, 18),
    supply_pct Decimal(38, 18),
    total_transaction_fees_sol Decimal(38, 18),
    total_priority_fees_sol Decimal(38, 18),
    total_validator_tips_sol Decimal(38, 18),
    total_dex_fees_sol Decimal(38, 18),
    total_platform_fees_sol Decimal(38, 18),
    total_fees_paid_sol Decimal(38, 18)
)
ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats:4222',
    nats_subjects = 'solana.position_overview',
    nats_format = 'RowBinary',
    nats_num_consumers = 4;

-- Fan-out to current state and history
CREATE MATERIALIZED VIEW datastreams.position_overview_current_mv
TO datastreams.position_overview AS
SELECT *, now64(9) AS last_updated
FROM datastreams.position_overview_queue;

CREATE MATERIALIZED VIEW datastreams.position_overview_history_mv
TO datastreams.position_history AS
SELECT *, now64(9) AS last_updated
FROM datastreams.position_overview_queue;
```

---

## Position Tracking Theory

### The Challenge: Multiple Data Sources

Unlike traditional exchanges where positions only change through trades, DEX positions are affected by multiple event types:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Position Update Sources                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐     ┌─────────────────────────────────────────────┐│
│  │   TRADES (Swaps)    │     │              Position Change                ││
│  │                     │     │                                             ││
│  │  • BUY: +tokens     │ ──► │  holdings = bought - sold + received - sent ││
│  │  • SELL: -tokens    │     │                                             ││
│  │  • Affects cost     │     │  Affects:                                   ││
│  │    basis & PnL      │     │  • holdings_token                           ││
│  └─────────────────────┘     │  • realized_pnl (on sells)                  ││
│                              │  • supply_pct                               ││
│  ┌─────────────────────┐     │  • total_fees_paid                          ││
│  │   TRANSFERS         │     │                                             ││
│  │                     │     └─────────────────────────────────────────────┘│
│  │  • SEND: -tokens    │ ──►                                                │
│  │  • RECEIVE: +tokens │                                                    │
│  │  • No PnL impact    │     Note: Transfers do NOT affect cost basis      │
│  │    (same owner)     │     or PnL - tokens maintain their original       │
│  └─────────────────────┘     cost basis when transferred between wallets   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Aggregation Formula

Position holdings are calculated by combining all sources:

```
holdings_token = total_tokens_bought
               - total_tokens_sold
               + total_tokens_received
               - total_tokens_sent
```

This formula handles:
- **Pure traders**: Only buy/sell, no transfers
- **Airdrop recipients**: Receive tokens, then may sell
- **Whale wallets**: Transfer between their own wallets
- **Mixed activity**: Any combination of the above

---

## The PositionOverview Data Model

### Complete Field Inventory (28 Fields)

Our position tracking includes **28 fields** organized into logical groups:

#### Identity Fields (5 fields)
```
┌────────────────────────────────────────────────────────────────────────────┐
│                           IDENTITY FIELDS                                   │
├─────────────────────┬─────────────────┬────────────────────────────────────┤
│ Field               │ Type            │ Purpose                            │
├─────────────────────┼─────────────────┼────────────────────────────────────┤
│ trader              │ String          │ Wallet public key (PRIMARY KEY)    │
│ click_user_id       │ Nullable(Str)   │ Platform user ID (internal only)   │
│ token_mint          │ String          │ Token address (PRIMARY KEY)        │
│ token_name          │ String          │ Human-readable name                │
│ token_symbol        │ String          │ Trading symbol                     │
└─────────────────────┴─────────────────┴────────────────────────────────────┘
```

#### Quantity Fields (6 fields)
```
┌────────────────────────────────────────────────────────────────────────────┐
│                          QUANTITY FIELDS                                    │
├─────────────────────────┬──────────────────┬───────────────────────────────┤
│ Field                   │ Type             │ Purpose                       │
├─────────────────────────┼──────────────────┼───────────────────────────────┤
│ holdings_token          │ Decimal(38,18)   │ Current balance (derived)     │
│ total_tokens_bought     │ Decimal(38,18)   │ Sum of all buys               │
│ total_tokens_sold       │ Decimal(38,18)   │ Sum of all sells              │
│ total_tokens_received   │ Decimal(38,18)   │ Sum of incoming transfers     │
│ total_tokens_sent       │ Decimal(38,18)   │ Sum of outgoing transfers     │
│ initial_tokens_bought   │ Decimal(38,18)   │ First buy amount (immutable)  │
└─────────────────────────┴──────────────────┴───────────────────────────────┘
```

The `initial_tokens_bought` field enables the **"Sell Initial"** feature - a one-click action to sell exactly what was first purchased, useful for taking profits while keeping any additional accumulation.

#### Volume Fields (4 fields)
```
┌────────────────────────────────────────────────────────────────────────────┐
│                           VOLUME FIELDS                                     │
├─────────────────────────┬──────────────────┬───────────────────────────────┤
│ Field                   │ Type             │ Purpose                       │
├─────────────────────────┼──────────────────┼───────────────────────────────┤
│ total_spent_sol         │ Decimal(38,18)   │ SOL paid for all buys         │
│ total_spent_usd         │ Decimal(38,18)   │ USD value of all buys         │
│ total_received_sol      │ Decimal(38,18)   │ SOL received from sells       │
│ total_received_usd      │ Decimal(38,18)   │ USD value of sells            │
└─────────────────────────┴──────────────────┴───────────────────────────────┘
```

#### PnL Fields (3 fields)
```
┌────────────────────────────────────────────────────────────────────────────┐
│                             PNL FIELDS                                      │
├─────────────────────────┬──────────────────┬───────────────────────────────┤
│ Field                   │ Type             │ Purpose                       │
├─────────────────────────┼──────────────────┼───────────────────────────────┤
│ realized_pnl_sol        │ Decimal(38,18)   │ Locked-in profit/loss (SOL)   │
│ realized_pnl_usd        │ Decimal(38,18)   │ Locked-in profit/loss (USD)   │
│ supply_pct              │ Decimal(38,18)   │ % of total supply held        │
└─────────────────────────┴──────────────────┴───────────────────────────────┘
```

#### Fee Tracking Fields (6 fields)
```
┌────────────────────────────────────────────────────────────────────────────┐
│                          FEE TRACKING FIELDS                                │
├─────────────────────────────┬──────────────────┬───────────────────────────┤
│ Field                       │ Type             │ Accumulation Rule         │
├─────────────────────────────┼──────────────────┼───────────────────────────┤
│ total_transaction_fees_sol  │ Decimal(38,18)   │ Per transaction           │
│ total_priority_fees_sol     │ Decimal(38,18)   │ Per transaction (subset)  │
│ total_validator_tips_sol    │ Decimal(38,18)   │ Per transaction (Jito)    │
│ total_dex_fees_sol          │ Decimal(38,18)   │ Per hop (multi-hop swaps) │
│ total_platform_fees_sol     │ Decimal(38,18)   │ Per transaction (1st hop) │
│ total_fees_paid_sol         │ Decimal(38,18)   │ Sum of all fees           │
└─────────────────────────────┴──────────────────┴───────────────────────────┘
```

---

## Understanding Cost Basis Methods

### Why Cost Basis Matters

Cost basis determines how we calculate profit/loss when selling tokens. Different methods yield different PnL figures:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Cost Basis Method Comparison                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Example: Buy 100 tokens @ 1 SOL, Buy 100 tokens @ 2 SOL, Sell 50 tokens    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  FIFO (First In, First Out)                                             ││
│  │  ────────────────────────────                                           ││
│  │  Sell 50 from first batch (cost: 1 SOL each)                           ││
│  │  Cost basis for sold tokens: 50 × 1 = 50 SOL                           ││
│  │  If sold at 3 SOL each: PnL = (50 × 3) - 50 = +100 SOL                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  LIFO (Last In, First Out)                                              ││
│  │  ────────────────────────────                                           ││
│  │  Sell 50 from second batch (cost: 2 SOL each)                          ││
│  │  Cost basis for sold tokens: 50 × 2 = 100 SOL                          ││
│  │  If sold at 3 SOL each: PnL = (50 × 3) - 100 = +50 SOL                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  AVERAGE COST (Our Approach)                                            ││
│  │  ────────────────────────────                                           ││
│  │  Average cost: (100×1 + 100×2) / 200 = 1.5 SOL                         ││
│  │  Cost basis for sold tokens: 50 × 1.5 = 75 SOL                         ││
│  │  If sold at 3 SOL each: PnL = (50 × 3) - 75 = +75 SOL                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Our Implementation: Weighted Average Cost

We use **weighted average cost** because it:
1. Requires minimal state (just totals, not per-lot tracking)
2. Is intuitive for traders to understand
3. Handles transfers gracefully (cost basis transfers with tokens)
4. Is efficient to compute in real-time

The formula:
```
avg_cost = total_spent_sol / total_tokens_bought
```

---

## Realized vs Unrealized PnL

### The Two Types of PnL

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PnL Classification                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   REALIZED PnL                           UNREALIZED PnL                      │
│   ────────────                           ──────────────                      │
│   ┌─────────────────────────────────┐   ┌─────────────────────────────────┐ │
│   │                                 │   │                                 │ │
│   │  "Locked in" profit/loss       │   │  "Paper" profit/loss            │ │
│   │                                 │   │                                 │ │
│   │  Triggered when you SELL       │   │  Based on current market price  │ │
│   │                                 │   │                                 │ │
│   │  Formula:                       │   │  Formula:                       │ │
│   │  received - (cost × sold)       │   │  (holdings × price) - cost_basis│ │
│   │                                 │   │                                 │ │
│   │  Stored in: position_overview  │   │  Computed at query time         │ │
│   │  (pre-computed by DataStreams)  │   │  (requires price join)          │ │
│   │                                 │   │                                 │ │
│   └─────────────────────────────────┘   └─────────────────────────────────┘ │
│                                                                              │
│   Example:                               Example:                            │
│   • Bought 1000 tokens @ 0.001 SOL      • Still holding 500 tokens          │
│   • Sold 500 tokens @ 0.003 SOL         • Current price: 0.005 SOL          │
│   • Realized: 1.5 - 0.5 = +1.0 SOL      • Cost basis: 500 × 0.001 = 0.5 SOL │
│                                          • Value: 500 × 0.005 = 2.5 SOL      │
│                                          • Unrealized: 2.5 - 0.5 = +2.0 SOL  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Realized PnL Calculation in DataStreams

The stream processor computes realized PnL on every sell:

```
For Open Position (still holding tokens):
    realized_pnl = total_received - (avg_cost × total_tokens_sold)

For Closed Position (holdings = 0):
    realized_pnl = total_received - total_spent
```

---

## Fee Tracking Architecture

### The Five Fee Categories

Every Solana DEX trade involves multiple fee types:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Fee Decomposition Per Trade                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Trade: Swap 1 SOL → 1000 BONK (via Jupiter, 2-hop route)                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. Transaction Fee: 0.000005 SOL                                        │ │
│  │    └── Base network fee, always present                                │ │
│  │                                                                         │ │
│  │ 2. Priority Fee: 0.0001 SOL (included in transaction fee)              │ │
│  │    └── Extra fee for faster inclusion, optional but common              │ │
│  │                                                                         │ │
│  │ 3. Validator Tip (Jito): 0.001 SOL                                      │ │
│  │    └── MEV protection tip, separate from transaction fee               │ │
│  │                                                                         │ │
│  │ 4. DEX Fees: 0.003 SOL (0.15% × 2 hops)                                │ │
│  │    └── Accumulated per hop: Raydium fee + Orca fee                      │ │
│  │                                                                         │ │
│  │ 5. Platform Fee: 0.01 SOL (1% to trading bot)                          │ │
│  │    └── Only counted once (first hop), goes to BullX/Axiom/etc          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Total Fees: 0.000005 + 0.001 + 0.003 + 0.01 = 0.014005 SOL                 │
│  (Priority fee NOT added separately - it's inside transaction fee)          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Fee Accumulation Rules

Different fees accumulate differently across multi-hop swaps:

| Fee Type | Accumulation | Rationale |
|----------|--------------|-----------|
| Transaction Fee | Per TX | One fee per transaction regardless of hops |
| Priority Fee | Per TX | Subset of transaction fee |
| Validator Tip | Per TX | One tip per transaction |
| DEX Fee | **Per Hop** | Each DEX in the route charges fees |
| Platform Fee | Per TX (1st hop) | Bot fee only charged once |

---

## ReplacingMergeTree Deep Dive

### Why ReplacingMergeTree?

Position data has a key characteristic: **we only care about the current state**. We need:
- Automatic deduplication (latest version wins)
- Efficient updates without explicit DELETEs
- Fast point lookups by (trader, token_mint)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ReplacingMergeTree Mechanics                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Timeline: Position updates for trader ABC, token XYZ                        │
│                                                                              │
│  T1: Buy 100 tokens ──► Insert row (version: 1000000001)                    │
│  T2: Buy 50 more    ──► Insert row (version: 1000000002)                    │
│  T3: Sell 75        ──► Insert row (version: 1000000003)                    │
│                                                                              │
│  Before Merge:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Part 1: [ABC, XYZ, holdings=100, version=1000000001]                  │ │
│  │  Part 2: [ABC, XYZ, holdings=150, version=1000000002]                  │ │
│  │  Part 3: [ABC, XYZ, holdings=75, version=1000000003]                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  After Merge (background process):                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Merged Part: [ABC, XYZ, holdings=75, version=1000000003]              │ │
│  │               └── Only highest version retained                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Query with FINAL:                                                           │
│  SELECT * FROM position_overview FINAL WHERE trader='ABC' AND token='XYZ'   │
│  └── Returns ONLY the row with version=1000000003                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Version Column Strategy

The version column determines which row "wins" during deduplication:

```sql
-- Version is derived from last_updated timestamp (nanosecond precision)
last_updated DateTime64(9) DEFAULT now64(9),
version UInt64 MATERIALIZED toUnixTimestamp64Nano(last_updated),
```

### Table Definition with Optimization

```sql
CREATE TABLE datastreams.position_overview ON CLUSTER '{cluster}'
(
    -- Primary identifiers
    trader String CODEC(ZSTD(3)),
    click_user_id Nullable(String) CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),
    token_name String CODEC(ZSTD(3)),
    token_symbol String CODEC(ZSTD(3)),

    -- Quantities (high precision decimals)
    holdings_token Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_sent Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_received Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_bought Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_sold Decimal(38, 18) CODEC(Gorilla, LZ4),
    initial_tokens_bought Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- Volumes
    total_spent_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_spent_usd Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_received_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_received_usd Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- P&L
    realized_pnl_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    realized_pnl_usd Decimal(38, 18) CODEC(Gorilla, LZ4),
    supply_pct Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- Fee Tracking
    total_transaction_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_priority_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_validator_tips_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_dex_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_platform_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_fees_paid_sol Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- Versioning
    last_updated DateTime64(9) DEFAULT now64(9) CODEC(Delta, LZ4),
    version UInt64 MATERIALIZED toUnixTimestamp64Nano(last_updated),

    -- Data Skipping Indexes
    INDEX idx_token token_mint TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_user click_user_id TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_symbol token_symbol TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4,
    INDEX idx_holdings holdings_token TYPE minmax GRANULARITY 4
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/position_overview',
    '{replica}',
    version  -- Version column for deduplication
)
ORDER BY (trader, token_mint)
SETTINGS
    index_granularity = 8192,
    merge_with_ttl_timeout = 86400;
```

### ORDER BY Strategy: (trader, token_mint)

The ORDER BY is optimized for the most common query patterns:

| Query Pattern | Efficiency | Why |
|---------------|------------|-----|
| Get all positions for trader | Excellent | Data contiguous by trader |
| Get position for trader + token | Excellent | Direct lookup |
| Get all holders of token | Requires scan | Token not first in ORDER BY |
| Top holders by holdings | Requires scan | Not indexed |

---

## Current State vs Historical Patterns

### Dual Table Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Dual Table Pattern                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NATS: solana.position_overview                                              │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                   position_overview_queue                                ││
│  │                   (NATS Engine - Ephemeral)                              ││
│  └──────────────────────────┬────────────────────────────────────────────┬─┘│
│                             │                                            │  │
│              ┌──────────────┴──────────────┐          ┌─────────────────┴─┐│
│              │                             │          │                   ││
│              ▼                             ▼          ▼                   ││
│  ┌─────────────────────────┐  ┌───────────────────────────────────────────┐│
│  │   position_overview     │  │        position_history                   ││
│  │   (ReplacingMergeTree)  │  │        (MergeTree)                        ││
│  │                         │  │                                           ││
│  │   • Current state only  │  │   • All snapshots                         ││
│  │   • Deduplicates on     │  │   • Partitioned by month                  ││
│  │     (trader, token)     │  │   • 90-day TTL                            ││
│  │   • Fast point lookups  │  │   • Time-series analysis                  ││
│  └─────────────────────────┘  └───────────────────────────────────────────┘│
│                                                                              │
│  Use Cases:                       Use Cases:                                 │
│  • "Show my positions"            • "Position history over time"            │
│  • "Top holders of token"         • "When did I buy/sell?"                  │
│  • "Current PnL"                  • "Holdings graph"                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### History Table Design

```sql
CREATE TABLE datastreams.position_history ON CLUSTER '{cluster}'
(
    -- Same fields as position_overview (without version)
    trader String CODEC(ZSTD(3)),
    click_user_id Nullable(String) CODEC(ZSTD(3)),
    token_mint String CODEC(ZSTD(3)),
    token_name String CODEC(ZSTD(3)),
    token_symbol String CODEC(ZSTD(3)),
    holdings_token Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_sent Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_received Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_bought Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_tokens_sold Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_spent_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_spent_usd Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_received_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_received_usd Decimal(38, 18) CODEC(Gorilla, LZ4),
    realized_pnl_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    realized_pnl_usd Decimal(38, 18) CODEC(Gorilla, LZ4),
    supply_pct Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- Fee tracking
    total_transaction_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_priority_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_validator_tips_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_dex_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_platform_fees_sol Decimal(38, 18) CODEC(Gorilla, LZ4),
    total_fees_paid_sol Decimal(38, 18) CODEC(Gorilla, LZ4),

    -- Timestamp (NOT version - we want all snapshots)
    last_updated DateTime64(9) CODEC(Delta, LZ4),

    -- Index for token lookups
    INDEX idx_token token_mint TYPE bloom_filter(0.01) GRANULARITY 4
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/position_history',
    '{replica}'
)
PARTITION BY toYYYYMM(last_updated)
ORDER BY (trader, token_mint, last_updated)
TTL last_updated + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

---

## Computed Fields Strategy

### What We Store vs What We Compute

Not all fields need to be stored. Some are more efficiently computed at query time:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Stored vs Computed Fields                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STORED (from stream)              COMPUTED (at query time)                  │
│  ────────────────────              ────────────────────────                  │
│                                                                              │
│  ✓ holdings_token                  ○ avg_cost_sol                            │
│  ✓ total_tokens_bought               = total_spent_sol / total_tokens_bought│
│  ✓ total_tokens_sold                                                         │
│  ✓ total_spent_sol                 ○ avg_exit_price_sol                      │
│  ✓ total_received_sol                = total_received_sol / total_tokens_sold│
│  ✓ realized_pnl_sol                                                          │
│                                    ○ cost_basis_sol                          │
│  Why store these?                    = holdings × avg_cost                   │
│  • Updated per trade                                                         │
│  • Base for all calcs             ○ unrealized_pnl_sol                       │
│  • Need for PnL                     = current_value - cost_basis             │
│                                      (requires price JOIN)                   │
│                                                                              │
│                                    ○ position_status                         │
│                                      = if(holdings > 0, 'OPEN', 'CLOSED')   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## View Architecture for PnL

### View 1: Computed Fields

```sql
CREATE VIEW datastreams.v_position_overview AS
SELECT
    -- Pass through all stored fields
    trader, click_user_id, token_mint, token_name, token_symbol,
    holdings_token, total_tokens_sent, total_tokens_received,
    total_tokens_bought, total_tokens_sold, initial_tokens_bought,
    total_spent_sol, total_spent_usd, total_received_sol, total_received_usd,
    realized_pnl_sol, realized_pnl_usd, supply_pct, last_updated,
    total_transaction_fees_sol, total_priority_fees_sol,
    total_validator_tips_sol, total_dex_fees_sol,
    total_platform_fees_sol, total_fees_paid_sol,

    -- COMPUTED: Average costs
    if(total_tokens_bought > 0,
       total_spent_sol / total_tokens_bought,
       toDecimal128(0, 18)) AS avg_cost_sol,
    if(total_tokens_bought > 0,
       total_spent_usd / total_tokens_bought,
       toDecimal128(0, 18)) AS avg_cost_usd,

    -- COMPUTED: Cost basis
    if(total_tokens_bought > 0,
       holdings_token * (total_spent_sol / total_tokens_bought),
       toDecimal128(0, 18)) AS cost_basis_sol,
    if(total_tokens_bought > 0,
       holdings_token * (total_spent_usd / total_tokens_bought),
       toDecimal128(0, 18)) AS cost_basis_usd,

    -- COMPUTED: Position status
    if(holdings_token > 0, 'OPEN', 'CLOSED') AS position_status

FROM datastreams.position_overview FINAL;
```

### View 2: With Prices and Full PnL

```sql
CREATE VIEW datastreams.v_position_with_prices AS
SELECT
    p.*,

    -- FROM TOKEN_METRICS (via JOIN)
    m.price_sol AS current_price_sol,
    m.price_usd AS current_price_usd,
    m.market_cap_usd,
    m.liquidity_usd,

    -- COMPUTED: Current value
    p.holdings_token * m.price_sol AS current_value_sol,
    p.holdings_token * m.price_usd AS current_value_usd,

    -- COMPUTED: Unrealized PnL
    (p.holdings_token * m.price_sol) - p.cost_basis_sol AS unrealized_pnl_sol,
    (p.holdings_token * m.price_usd) - p.cost_basis_usd AS unrealized_pnl_usd,

    -- COMPUTED: Unrealized PnL percentage
    if(p.cost_basis_sol > 0,
       ((p.holdings_token * m.price_sol) - p.cost_basis_sol) / p.cost_basis_sol * 100,
       toDecimal128(0, 18)) AS unrealized_pnl_pct,

    -- COMPUTED: Total PnL (realized + unrealized)
    p.realized_pnl_sol + ((p.holdings_token * m.price_sol) - p.cost_basis_sol) AS total_pnl_sol,
    p.realized_pnl_usd + ((p.holdings_token * m.price_usd) - p.cost_basis_usd) AS total_pnl_usd

FROM datastreams.v_position_overview AS p
LEFT JOIN datastreams.token_metrics FINAL AS m ON p.token_mint = m.token_mint;
```

---

## Query Optimization Patterns

### Common Query Patterns

#### 1. Get All Positions for Trader

```sql
-- Fast: Uses ORDER BY prefix
SELECT
    token_symbol,
    holdings_token,
    current_value_usd,
    unrealized_pnl_pct,
    total_pnl_usd
FROM datastreams.v_position_with_prices
WHERE trader = 'ABC123...'
  AND holdings_token > 0  -- Only open positions
ORDER BY current_value_usd DESC;
```

#### 2. Portfolio Summary

```sql
SELECT
    trader,
    count() AS positions_count,
    countIf(position_status = 'OPEN') AS open_positions,
    sum(current_value_usd) AS total_value_usd,
    sum(realized_pnl_usd) AS total_realized_usd,
    sum(unrealized_pnl_usd) AS total_unrealized_usd,
    sum(total_pnl_usd) AS total_pnl_usd,
    sum(total_fees_paid_sol) AS total_fees_sol
FROM datastreams.v_position_with_prices
WHERE trader = 'ABC123...'
GROUP BY trader;
```

#### 3. Top Holders of Token

```sql
-- Uses bloom_filter index on token_mint
SELECT
    trader,
    holdings_token,
    supply_pct,
    total_spent_sol,
    realized_pnl_sol
FROM datastreams.v_position_overview
WHERE token_mint = 'XYZ789...'
  AND holdings_token > 0
ORDER BY holdings_token DESC
LIMIT 100;
```

#### 4. Fee Analysis

```sql
-- Breakdown of fees for a trader
SELECT
    trader,
    sum(total_transaction_fees_sol) AS total_tx_fees,
    sum(total_priority_fees_sol) AS total_priority,
    sum(total_validator_tips_sol) AS total_jito_tips,
    sum(total_dex_fees_sol) AS total_dex_fees,
    sum(total_platform_fees_sol) AS total_platform_fees,
    sum(total_fees_paid_sol) AS grand_total_fees
FROM datastreams.position_overview FINAL
WHERE trader = 'ABC123...'
GROUP BY trader;
```

---

## Summary

### Position Tracking Architecture

In this article, we've covered the complete position tracking system:

| Component | Purpose | Engine/Strategy |
|-----------|---------|-----------------|
| **position_overview** | Current state | ReplacingMergeTree with version column |
| **position_history** | Historical snapshots | MergeTree with 90-day TTL |
| **v_position_overview** | Computed fields | View with FINAL |
| **v_position_with_prices** | Full PnL | View with token_metrics JOIN |

### Data Pipeline Summary

| Stage | Component | Description |
|-------|-----------|-------------|
| **Source** | solana.trades.v3, solana.transfers.v2 | Raw trade and transfer events |
| **Transform** | position_overview_from_trades/send/receive | Stateful position updates |
| **State** | PositionState | Per (trader, token) position state |
| **Output** | solana.position_overview | 28-field position updates |
| **Ingestion** | NATS Engine + MVs | Fan-out to current + history |
| **Query** | Views with JOINs | Computed fields + prices |

### Key Design Decisions

1. **Weighted Average Cost** - Simple, efficient, handles transfers gracefully
2. **Realized PnL in Stream** - Computed when trades occur, stored for fast access
3. **Unrealized PnL at Query** - Requires current price, computed via JOIN
4. **Dual Table Pattern** - Current state + history for different use cases
5. **Fee Decomposition** - Five categories with different accumulation rules

### Metrics Covered

| Category | Fields | Description |
|----------|--------|-------------|
| Identity | 5 | trader, token, names |
| Quantities | 6 | holdings, bought, sold, transfers |
| Volumes | 4 | spent/received in SOL/USD |
| PnL | 3 | realized PnL, supply % |
| Fees | 6 | All fee categories |
| **Total Stored** | **28** | **Fields from DataStreams** |
| **Computed** | ~12 | avg_cost, unrealized_pnl, etc. |

---

## What's Next

In **Part 5**, we'll explore **Discovery & Trending Analytics** - how we identify new tokens, track graduation events, and build the trending page metrics that help traders find opportunities.

---

## Implementation References

| Document | Content |
|----------|---------|
| [11_POSITION_OVERVIEW_SETUP.md](./11_POSITION_OVERVIEW_SETUP.md) | Complete Position DDL |
| [08_ENRICHED_TRADES_SETUP.md](./08_ENRICHED_TRADES_SETUP.md) | Trade struct with fees |
| [17_DATA_FLOW.md](./17_DATA_FLOW.md) | Pipeline architecture |

---

*Part 4 of 6 in the "Building Real-Time Solana DEX Analytics with ClickHouse" series.*

- [Part 1: Architecture Overview](./BLOG_01_ARCHITECTURE_OVERVIEW.md)
- [Part 2: Trade Enrichment & OHLCV](./BLOG_02_TRADE_ENRICHMENT_OHLCV.md)
- [Part 3: Token Metrics & Rolling Windows](./BLOG_03_TOKEN_METRICS_ROLLING_WINDOWS.md)
- **Part 4: Position Tracking & PnL** (You are here)
- Part 5: Discovery & Trending Analytics (Coming soon)
- Part 6: Production Patterns & Monitoring (Coming soon)
