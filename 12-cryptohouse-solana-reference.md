# CryptoHouse Solana Reference

## Source

- **Repository**: [ClickHouse/CryptoHouse](https://github.com/ClickHouse/CryptoHouse)
- **Live Instance**: `clickhouse client --host crypto-clickhouse.clickhouse.com --secure --user crypto`
- **Use Case**: Production Solana blockchain analytics with Ethereum and Polymarket data

---

## 1. Schema Design

### 1.1 Core Tables Overview

| Table | Engine | Primary Key | Purpose |
|-------|--------|-------------|---------|
| `solana.blocks` | ReplacingMergeTree | (block_timestamp, slot) | Block metadata |
| `solana.transactions` | ReplacingMergeTree | (block_timestamp, block_slot) | All transactions |
| `solana.accounts` | ReplacingMergeTree | (block_timestamp, block_slot) | Account state snapshots |
| `solana.tokens` | ReplacingMergeTree | (block_timestamp, block_slot) | Token/NFT metadata |
| `solana.token_transfers` | ReplacingMergeTree | (block_timestamp, block_slot) | SPL token transfers |
| `solana.block_rewards` | ReplacingMergeTree | (block_timestamp, block_slot) | Validator rewards |
| `solana.instructions` | ReplacingMergeTree | (block_timestamp, block_slot) | Program instructions |

### 1.2 Blocks Table

```sql
CREATE TABLE solana.blocks
(
    block_hash String,
    block_timestamp DateTime64(6),      -- Microsecond precision
    height Int64,
    leader String,                       -- Validator pubkey
    leader_reward Decimal(38, 9),        -- 9 decimals for lamports
    previous_block_hash String,
    slot Int64,
    transaction_count Int64
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, slot)
ORDER BY (block_timestamp, slot, block_hash)
```

**Key Design Decisions:**
- `DateTime64(6)` - Microsecond precision for Solana's ~400ms block times
- `Decimal(38, 9)` - 9 decimal places matches SOL's lamport precision (1 SOL = 1B lamports)
- Primary Key on `(block_timestamp, slot)` - Time-first for range queries, slot for uniqueness

### 1.3 Transactions Table

```sql
CREATE TABLE solana.transactions
(
    -- Nested account data as Array of Tuples
    accounts Array(Tuple(
        pubkey String,
        signer Bool,
        writable Bool
    )),

    -- Balance changes for all accounts in transaction
    balance_changes Array(Tuple(
        account String,
        before Decimal(38, 9),
        after Decimal(38, 9)
    )),

    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,                          -- JSON error or 'null'
    fee Decimal(38, 9),
    index Int64,                         -- Transaction index in block
    log_messages Array(String),          -- Program logs

    -- Token balance snapshots
    post_token_balances Array(Tuple(
        account_index Int64,
        mint String,
        owner String,
        amount Decimal(76, 38),          -- High precision for tokens
        decimals Int64
    )),
    pre_token_balances Array(Tuple(
        account_index Int64,
        mint String,
        owner String,
        amount Decimal(76, 38),
        decimals Int64
    )),

    recent_block_hash String,
    signature String,                    -- Transaction signature (unique ID)
    status String                        -- 'Success' or 'Fail'
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, signature)
```

**Key Design Decisions:**
- `Array(Tuple(...))` - Nested structures for related data (accounts, balances)
- `Decimal(76, 38)` - 38 decimal places for token amounts (handles any token precision)
- `log_messages Array(String)` - Preserves program execution logs for debugging
- `err String` - JSON string allows flexible error parsing

### 1.4 Accounts Table

```sql
CREATE TABLE solana.accounts
(
    account_type String,

    -- Validator-specific fields
    authorized_voters Array(Tuple(authorized_voter String, epoch Int64)),
    authorized_withdrawer String,
    commission Int64,
    epoch_credits Array(Tuple(credits String, epoch Int64, previous_credits String)),
    votes Array(Tuple(confirmation_count Int64, slot Int64)),

    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),

    -- Account data (base64 encoded)
    data Array(Tuple(raw String, encoding String)),

    executable Bool,
    is_native Bool,
    lamports Decimal(38, 9),
    last_timestamp Array(Tuple(slot Int64, timestamp DateTime64(6))),

    -- Token account fields
    mint String,
    token_amount Decimal(38, 9),
    token_amount_decimals Int64,

    node_pubkey String,
    owner String,
    prior_voters Array(Tuple(authorized_pubkey String, epoch_of_last_authorized_switch Int64, target_epoch Int64)),
    program String,
    program_data String,
    pubkey String,
    rent_epoch Int64,
    retrieval_timestamp DateTime64(6),
    root_slot Int64,
    space Int64,
    state String,
    tx_signature String
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, pubkey, block_hash)
```

**Key Design Decisions:**
- Polymorphic schema - handles regular accounts, token accounts, and validator accounts
- `Array(Tuple(...))` for validator voting data (epoch credits, votes)
- ORDER BY includes `pubkey` for efficient account lookups

### 1.5 Tokens Table (SPL Tokens + NFTs)

```sql
CREATE TABLE solana.tokens
(
    block_slot Int64,
    block_hash String,
    block_timestamp DateTime64(6),
    tx_signature String,
    retrieval_timestamp DateTime64(6),

    is_nft Bool,                         -- NFT flag
    mint String,                         -- Token mint address
    update_authority String,
    name String,
    symbol String,
    uri String,                          -- Metadata URI
    seller_fee_basis_points Decimal(38, 9),

    -- NFT creators with verification status
    creators Array(Tuple(
        address String,
        verified Bool,
        share Int64
    )),

    primary_sale_happened Bool,
    is_mutable Bool
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, tx_signature, symbol)
```

### 1.6 Token Transfers Table

```sql
CREATE TABLE solana.token_transfers
(
    authority String,
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    decimals Decimal(38, 9),
    destination String,
    fee Decimal(38, 9),
    fee_decimals Decimal(38, 9),
    memo String,
    mint String,
    mint_authority String,
    source String,
    transfer_type String,                -- 'transfer', 'transferChecked', etc.
    tx_signature String,
    value Decimal(38, 9),

    id String DEFAULT generateULID()     -- Auto-generated unique ID
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, id)
```

**Key Pattern: `generateULID()`**
- Generates sortable unique IDs (better than UUID for time-series)
- Used when natural key isn't available or unique enough

### 1.7 Block Rewards Table

```sql
CREATE TABLE solana.block_rewards
(
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    commission Decimal(38, 9),
    lamports Decimal(38, 9),
    post_balance Decimal(38, 9),
    pubkey String,                       -- Validator pubkey
    reward_type String                   -- 'Fee', 'Rent', 'Staking', 'Voting'
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, pubkey)
```

---

## 2. Data Types for Crypto

### 2.1 Precision Guidelines

| Data Type | Decimals | Use Case | Example |
|-----------|----------|----------|---------|
| `Decimal(38, 9)` | 9 | SOL amounts (lamports) | 1.234567890 SOL |
| `Decimal(76, 38)` | 38 | Token amounts (any precision) | Handles SHIB, etc. |
| `DateTime64(6)` | microseconds | Block/tx timestamps | Solana's ~400ms blocks |
| `DateTime64(9)` | nanoseconds | High-frequency data | Order execution times |
| `Int64` | - | Slots, epochs, indices | Block slot numbers |
| `String` | - | Addresses, signatures | Base58 encoded keys |
| `Bool` | - | Flags | is_nft, signer, writable |

### 2.2 Why Decimal over Float?

```sql
-- WRONG: Float loses precision
CREATE TABLE bad_example (amount Float64)  -- 0.1 + 0.2 != 0.3

-- CORRECT: Decimal preserves exact values
CREATE TABLE good_example (amount Decimal(38, 9))  -- Exact arithmetic
```

Financial data requires exact decimal arithmetic. ClickHouse's Decimal type provides this.

### 2.3 Array(Tuple) for Nested Blockchain Data

```sql
-- Pattern: Nested structures without JSON overhead
balance_changes Array(Tuple(
    account String,
    before Decimal(38, 9),
    after Decimal(38, 9)
))

-- Access nested fields
SELECT
    balance_changes[1].account AS first_account,
    balance_changes[1].after - balance_changes[1].before AS change
FROM solana.transactions
```

---

## 3. Staging Pattern (Null Engine + MV)

### 3.1 Why Staging Tables?

When ingesting from external sources (Kafka, APIs), data often arrives as strings. The staging pattern:
1. Accepts raw string data (no parsing errors on insert)
2. Transforms via Materialized View
3. Inserts typed data into main table
4. Staging table stores nothing (Null engine)

### 3.2 Stage Transactions Example

```sql
-- Staging table: accepts strings, stores nothing
CREATE TABLE solana.stage_transactions
(
    accounts String,                     -- JSON string
    balance_changes String,              -- JSON string
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,
    fee Decimal(38, 9),
    index Int64,
    log_messages String,                 -- JSON array as string
    pre_token_balances String,
    post_token_balances String,
    recent_block_hash String,
    signature String,
    status String
)
ENGINE = Null  -- Doesn't store data, just passes to MV
```

### 3.3 Transform MV

```sql
CREATE MATERIALIZED VIEW solana.stage_transactions_mv
TO solana.transactions
AS SELECT
    -- Cast JSON strings to typed Arrays
    CAST(accounts, 'Array(Tuple(String, Bool, Bool))') AS accounts,
    CAST(balance_changes, 'Array(Tuple(String, Decimal(38, 9), Decimal(38, 9)))') AS balance_changes,

    block_hash,
    block_slot,
    block_timestamp,
    compute_units_consumed,
    err,
    fee,
    index,

    -- Parse JSON array
    JSONExtract(log_messages, 'Array(String)') AS log_messages,

    CAST(pre_token_balances, 'Array(Tuple(Int64, String, String, Decimal(76, 38), Int64))') AS pre_token_balances,
    CAST(post_token_balances, 'Array(Tuple(Int64, String, String, Decimal(76, 38), Int64))') AS post_token_balances,

    recent_block_hash,
    signature,
    status
FROM solana.stage_transactions
```

### 3.4 Stage Accounts Example

```sql
CREATE TABLE solana.stage_accounts
(
    account_type String,
    authorized_voters String,            -- JSON string
    authorized_withdrawer String,
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    commission Int64,
    data String,                         -- JSON string
    epoch_credits String,                -- JSON string
    executable Bool,
    is_native Bool,
    lamports Decimal(38, 9),
    last_timestamp String,               -- JSON string
    mint String,
    node_pubkey String,
    owner String,
    prior_voters String,                 -- JSON string
    program String,
    program_data String,
    pubkey String,
    rent_epoch Int64,
    retrieval_timestamp DateTime64(6),
    root_slot Int64,
    space Int64,
    state String,
    token_amount Decimal(38, 9),
    token_amount_decimals Int64,
    tx_signature String,
    votes String                         -- JSON string
)
ENGINE = Null

CREATE MATERIALIZED VIEW solana.stage_accounts_mv
TO solana.accounts
AS SELECT
    account_type,
    CAST(authorized_voters, 'Array(Tuple(String, Int64))') AS authorized_voters,
    authorized_withdrawer,
    block_hash,
    block_slot,
    block_timestamp,
    commission,
    JSONExtract(concat('[', data, ']'), 'Array(Tuple(String, String))') AS data,
    CAST(epoch_credits, 'Array(Tuple(String, Int64, String))') AS epoch_credits,
    executable,
    is_native,
    lamports,
    CAST(last_timestamp, 'Array(Tuple(Int64, DateTime64(6)))') AS last_timestamp,
    mint,
    node_pubkey,
    owner,
    CAST(prior_voters, 'Array(Tuple(String, Int64, Int64))') AS prior_voters,
    program,
    program_data,
    pubkey,
    rent_epoch,
    retrieval_timestamp,
    root_slot,
    space,
    state,
    token_amount,
    token_amount_decimals,
    tx_signature,
    CAST(votes, 'Array(Tuple(Int64, Int64))') AS votes
FROM solana.stage_accounts
```

---

## 4. Filtering Patterns

### 4.1 Filter Vote Transactions

Solana has ~70% vote transactions. Filter them for analytics:

```sql
-- Destination table for non-voting transactions
CREATE TABLE solana.transactions_non_voting
(
    -- Same schema as transactions
    accounts Array(Tuple(pubkey String, signer Bool, writable Bool)),
    balance_changes Array(Tuple(account String, before Decimal(38, 9), after Decimal(38, 9))),
    block_hash String,
    block_slot Int64,
    block_timestamp DateTime64(6),
    compute_units_consumed Decimal(38, 9),
    err String,
    fee Decimal(38, 9),
    index Int64,
    log_messages Array(String),
    post_token_balances Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    pre_token_balances Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    recent_block_hash String,
    signature String,
    status String
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, signature)

-- MV with arrayExists filter
CREATE MATERIALIZED VIEW solana.transactions_non_voting_mv
TO solana.transactions_non_voting
AS SELECT *
FROM solana.transactions
WHERE NOT arrayExists(
    account -> account.1 = 'Vote111111111111111111111111111111111111111',
    accounts
)
```

**Key Pattern: `arrayExists` with Lambda**
```sql
-- Check if any account matches condition
arrayExists(
    account -> account.1 = 'Vote111111111111111111111111111111111111111',
    accounts
)

-- Returns true if Vote program is in accounts array
```

---

## 5. Aggregation Tables

### 5.1 Daily Block & Transaction Counts

```sql
CREATE TABLE solana.block_txn_counts_by_day
(
    day Date,
    block_count AggregateFunction(uniqCombined(14), String),
    txn_count AggregateFunction(sum, Int64)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.block_txn_counts_by_day_mv
TO solana.block_txn_counts_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    uniqCombinedState(14)(block_hash) AS block_count,  -- HyperLogLog state
    sumState(transaction_count) AS txn_count           -- Sum state
FROM solana.blocks
GROUP BY day
```

**Query Pattern:**
```sql
SELECT
    day,
    uniqCombinedMerge(14)(block_count) AS block_count,
    sumMerge(txn_count) AS txn_count
FROM solana.block_txn_counts_by_day
GROUP BY day
ORDER BY day DESC
```

### 5.2 Daily Fees & Signer Counts

```sql
CREATE TABLE solana.daily_fees_by_day
(
    day DateTime,
    avg_fee_sol AggregateFunction(avg, Float64),
    fee_sol AggregateFunction(sum, Float64),
    signer_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.daily_fees_by_day_mv
TO solana.daily_fees_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    avgState(fee / 1e9) AS avg_fee_sol,              -- Convert lamports to SOL
    sumState(fee / 1e9) AS fee_sol,
    uniqCombinedState(14)(accounts[1].1) AS signer_count  -- First account is signer
FROM solana.transactions_non_voting
GROUP BY day
```

### 5.3 Token Transfer Metrics

```sql
CREATE TABLE solana.token_transfer_metrics_by_day
(
    day DateTime,
    transfer_count AggregateFunction(uniqCombined(14), String),
    sender_count AggregateFunction(uniqCombined(14), String),
    reciever_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY day

CREATE MATERIALIZED VIEW solana.token_transfer_metrics_by_day_mv
TO solana.token_transfer_metrics_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    uniqCombinedState(14)(tx_signature) AS transfer_count,
    uniqCombinedState(14)(source) AS sender_count,
    uniqCombinedState(14)(destination) AS reciever_count
FROM solana.token_transfers
GROUP BY day
```

### 5.4 Metrics by Type (Multi-Dimensional)

```sql
CREATE TABLE solana.token_transfer_metrics_by_type_by_day
(
    day DateTime,
    transfer_type LowCardinality(String),
    transfer_count AggregateFunction(uniqCombined(14), String),
    sender_count AggregateFunction(uniqCombined(14), String),
    reciever_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY (day, transfer_type)

CREATE MATERIALIZED VIEW solana.token_transfer_metrics_by_type_by_day_mv
TO solana.token_transfer_metrics_by_type_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    transfer_type,
    uniqCombinedState(14)(tx_signature) AS transfer_count,
    uniqCombinedState(14)(source) AS sender_count,
    uniqCombinedState(14)(destination) AS reciever_count
FROM solana.token_transfers
GROUP BY day, transfer_type
```

### 5.5 Simple Counter Tables (SummingMergeTree)

```sql
-- When you just need counts, SummingMergeTree is simpler
CREATE TABLE solana.transactions_per_day
(
    date Date,
    count Int64
)
ENGINE = SummingMergeTree
ORDER BY date

CREATE MATERIALIZED VIEW solana.transactions_per_day_mv
TO solana.transactions_per_day
AS SELECT
    block_timestamp::Date AS date,
    count() AS count
FROM solana.transactions
GROUP BY date
```

**Query (automatic summing on merge):**
```sql
SELECT date, sum(count) AS count
FROM solana.transactions_per_day
GROUP BY date
```

### 5.6 Top Token Creators

```sql
CREATE TABLE solana.top_token_creators_by_day
(
    day DateTime,
    creator_address String,
    token_count AggregateFunction(uniqCombined(14), String)
)
ENGINE = AggregatingMergeTree
ORDER BY (day, creator_address)

CREATE MATERIALIZED VIEW solana.top_token_creators_by_day_mv
TO solana.top_token_creators_by_day
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    creators[1].1 AS creator_address,  -- First creator's address
    uniqCombinedState(14)(mint) AS token_count
FROM solana.tokens
GROUP BY day, creator_address
```

---

## 6. Query Patterns

### 6.1 ARRAY JOIN for Log Parsing

```sql
-- Parse compute units from transaction logs
SELECT
    signature AS id,
    maxIf(
        toInt64OrNull(splitByString(' ', log)[4]),
        log LIKE 'Program % consumed % of % compute units'
    ) AS computed_units_spent
FROM solana.transactions
ARRAY JOIN log_messages AS log
WHERE block_slot = 293312554
GROUP BY id
```

### 6.2 Balance Change Analysis

```sql
-- Most active account by balance change
SELECT
    account,
    sum(abs(before - after)) AS absolute_balance_change
FROM (
    SELECT
        arrayJoin(balance_changes) AS balance_change,
        balance_change.1 AS account,
        balance_change.2 AS before,
        balance_change.3 AS after
    FROM solana.transactions_non_voting
    WHERE toDate(block_timestamp) = '2024-07-24'
)
GROUP BY account
ORDER BY absolute_balance_change DESC
```

### 6.3 Compute Unit Analysis with CTEs

```sql
WITH
-- Extract requested compute units from ComputeBudget instruction
computebudget_a AS (
    SELECT
        tx_signature AS id,
        CASE
            WHEN substring(base58Decode(data), 1, 1) = toFixedString('\x02', 1)
            THEN reinterpretAsUInt32(substring(base58Decode(data), 2, 3))
            ELSE 0
        END AS requestedUnits
    FROM solana.instructions
    WHERE block_slot = 293312554
      AND program_id = 'ComputeBudget111111111111111111111111111111'
),
-- Deduplicate (take highest requested)
computebudget AS (
    SELECT id, requestedUnits,
           ROW_NUMBER() OVER (PARTITION BY id ORDER BY requestedUnits DESC) AS rn
    FROM computebudget_a
),
-- Extract consumed units from logs
consumedunits_a AS (
    SELECT
        signature AS id,
        maxIf(
            toInt64OrNull(splitByString(' ', log)[4]),
            log LIKE 'Program % consumed % of % compute units'
        ) AS computed_units_spent
    FROM solana.transactions
    ARRAY JOIN log_messages AS log
    WHERE block_slot = 293312554
    GROUP BY id
),
consumedunits AS (
    SELECT id, MAX(computed_units_spent) AS consumed_units
    FROM consumedunits_a
    GROUP BY id
)
SELECT
    concat('https://explorer.solana.com/tx/', t.signature) AS explorer_link,
    signature,
    CASE WHEN t.err = 'null' THEN '✅' ELSE '❌' END AS success,
    COALESCE(cb.requestedUnits, 0) - COALESCE(cu.consumed_units, 0) AS excessComputeUnit,
    COALESCE(cb.requestedUnits, 0) AS requestedUnits,
    COALESCE(cu.consumed_units, 0) AS consumedUnits
FROM solana.transactions t
LEFT JOIN computebudget cb ON t.signature = cb.id
LEFT JOIN consumedunits cu ON t.signature = cu.id
WHERE block_slot = 293312554 AND cb.rn = 1
ORDER BY excessComputeUnit DESC
```

### 6.4 OHLC Candlestick Query (Polymarket Example)

```sql
WITH prices AS (
    SELECT
        toStartOfInterval(e.timestamp, INTERVAL 10 second) AS time_interval,
        CASE
            WHEN pos.id IN ('21742633...', '69236923...')
            THEN e.maker_amount_filled / e.taker_amount_filled
            ELSE 1 - (e.maker_amount_filled / e.taker_amount_filled)
        END AS price
    FROM polymarket.orders_filled e
    INNER JOIN polymarket.token_id_condition pos
        ON e.maker_asset_id = pos.id OR e.taker_asset_id = pos.id
    WHERE pos.condition = '0xdd22472e...'
      AND maker_asset_id = '0'
      AND timestamp < '2024-11-05'::Date32
),
filled_prices AS (
    SELECT
        time_interval,
        price,
        IF(price = 0,
           anyLast(price) OVER (ORDER BY time_interval),
           price
        ) AS filled_price
    FROM prices
)
SELECT
    time_interval,
    MIN(filled_price) AS low_price,
    MAX(filled_price) AS high_price,
    any(filled_price) AS open_price,
    anyLast(filled_price) AS close_price
FROM filled_prices
GROUP BY time_interval
ORDER BY time_interval DESC
```

### 6.5 Window Functions for Rolling Averages

```sql
SELECT
    toStartOfDay(block_timestamp) AS day,
    count() AS txns,
    ROUND(AVG(txns) OVER (
        ORDER BY day ASC
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    )) AS `7d_avg`
FROM base.transactions
WHERE day > '2023-07-12'
GROUP BY day
ORDER BY day DESC
```

### 6.6 Cumulative Totals with Window Functions

```sql
SELECT
    toStartOfDay(min_date) AS day,
    count() AS new_users,
    sum(new_users) OVER (ORDER BY day ASC) AS total_addresses
FROM ethereum.first_time_for_address
WHERE day > '2023-07-12'
GROUP BY day
ORDER BY day ASC
```

---

## 7. Security Model

### 7.1 Read-Only User with Limits

```sql
CREATE USER crypto
IDENTIFIED WITH double_sha1_password BY 'hash...'
DEFAULT ROLE NONE
SETTINGS
    readonly = 1,
    add_http_cors_header = true,
    max_execution_time = 60,
    max_rows_to_read = 10000000000,
    max_bytes_to_read = 1000000000000,
    max_network_bandwidth = 25000000,
    max_memory_usage = 20000000000,
    max_bytes_before_external_group_by = 10000000000,
    max_result_rows = 10000,
    max_result_bytes = 10000000,
    result_overflow_mode = 'throw',
    read_overflow_mode = 'throw'
```

### 7.2 Settings Profile with Query Cache

```sql
CREATE SETTINGS PROFILE crypto
SETTINGS
    readonly = 1,
    add_http_cors_header = true,
    max_execution_time = 60,
    max_rows_to_read = 10000000000,
    max_bytes_to_read = 1000000000000,
    max_network_bandwidth = 25000000,
    max_memory_usage = 20000000000,
    max_bytes_before_external_group_by = 10000000000,
    max_result_rows = 1000,
    max_result_bytes = 10000000,
    result_overflow_mode = 'break' CHANGEABLE_IN_READONLY,
    read_overflow_mode = 'break' CHANGEABLE_IN_READONLY,
    enable_http_compression = true,
    use_query_cache = true,                              -- Enable query caching
    query_cache_nondeterministic_function_handling = 'save' CHANGEABLE_IN_READONLY
TO crypto
```

### 7.3 Granular Permissions

```sql
-- Database-level grants
GRANT SELECT ON solana.* TO crypto;
GRANT SELECT ON ethereum.* TO crypto;
GRANT SELECT ON polymarket.* TO crypto;

-- Table-level grants
GRANT SELECT ON default.queries TO crypto;

-- Column-level grants (monitor user)
GRANT SELECT(elapsed, initial_user, query_id, read_bytes, read_rows, total_rows_approx)
ON system.processes TO monitor;

-- Special permissions
GRANT KILL QUERY ON *.* TO monitor_admin;
```

### 7.4 IP-Based Quotas

```sql
CREATE QUOTA crypto
KEYED BY ip_address
FOR INTERVAL 1 hour MAX
    queries = 60,
    result_rows = 3000000000,
    read_rows = 3000000000000,
    execution_time = 6000
TO crypto;

CREATE QUOTA monitor
KEYED BY ip_address
FOR INTERVAL 1 hour MAX queries = 1200
TO monitor;
```

---

## 8. Click Adaptation Notes

### 8.1 Schema Adaptations for Click

| CryptoHouse | Click Equivalent | Notes |
|-------------|------------------|-------|
| `solana.transactions` | `trades` / `orders` | Adapt for exchange data |
| `solana.token_transfers` | `transfers` | SPL transfers map to token movements |
| `solana.blocks` | `blocks` | Keep for chain state tracking |
| `solana.accounts` | `positions` / `balances` | Account state snapshots |

### 8.2 Additional Tables Click Needs

```sql
-- OHLCV candles (not in CryptoHouse)
CREATE TABLE ohlcv (
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    interval LowCardinality(String),
    timestamp DateTime64(3),
    open Decimal(38, 18),
    high Decimal(38, 18),
    low Decimal(38, 18),
    close Decimal(38, 18),
    volume Decimal(38, 18)
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, exchange, interval, timestamp)

-- Liquidations
CREATE TABLE liquidations (
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    timestamp DateTime64(3),
    side LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    value_usd Decimal(38, 2)
)
ENGINE = MergeTree
ORDER BY (symbol, exchange, timestamp)

-- Funding rates
CREATE TABLE funding_rates (
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    timestamp DateTime64(3),
    funding_rate Decimal(38, 18),
    mark_price Decimal(38, 18),
    index_price Decimal(38, 18)
)
ENGINE = MergeTree
ORDER BY (symbol, exchange, timestamp)
```

### 8.3 Key Settings to Adopt

```sql
-- From CryptoHouse user settings
SET use_query_cache = true;                    -- Cache repeated queries
SET enable_http_compression = true;            -- Compress API responses
SET max_bytes_before_external_group_by = 10000000000;  -- Spill to disk for large GROUP BY
SET result_overflow_mode = 'break';            -- Graceful truncation
```

### 8.4 Staging Pattern for Click Ingestion

```sql
-- Stage table for Kafka/API ingestion
CREATE TABLE stage_trades (
    raw_data String
) ENGINE = Null;

-- Transform MV
CREATE MATERIALIZED VIEW stage_trades_mv TO trades AS
SELECT
    JSONExtractString(raw_data, 'symbol') AS symbol,
    JSONExtractString(raw_data, 'exchange') AS exchange,
    parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')) AS timestamp,
    JSONExtractFloat(raw_data, 'price')::Decimal(38, 18) AS price,
    JSONExtractFloat(raw_data, 'quantity')::Decimal(38, 18) AS quantity,
    JSONExtractString(raw_data, 'side') AS side
FROM stage_trades;
```

---

## 9. Functions Reference

### 9.1 Crypto-Specific Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `base58Decode(s)` | Decode Solana base58 data | `base58Decode(instruction_data)` |
| `generateULID()` | Sortable unique ID | `DEFAULT generateULID()` |
| `reinterpretAsUInt32(s)` | Binary to integer | Parse instruction bytes |
| `splitByString(sep, s)` | Parse log messages | `splitByString(' ', log)[4]` |

### 9.2 Time Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `toStartOfDay(ts)` | Daily aggregation | `toStartOfDay(block_timestamp)` |
| `toStartOfInterval(ts, INTERVAL)` | Custom intervals | `toStartOfInterval(ts, INTERVAL 10 second)` |
| `parseDateTimeBestEffort(s)` | Flexible parsing | Parse various timestamp formats |

### 9.3 Array Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `arrayExists(lambda, arr)` | Check condition | Filter vote transactions |
| `arrayJoin(arr)` | Unnest array | Expand balance_changes |
| `arr[n].field` | Access tuple field | `accounts[1].1` (first account pubkey) |

### 9.4 Aggregate State Functions

| Function | Use Case | Example |
|----------|----------|---------|
| `uniqCombinedState(14)(col)` | HyperLogLog state | Distinct count storage |
| `uniqCombinedMerge(14)(col)` | Merge HLL states | Query aggregate table |
| `sumState(col)` | Sum state | Store for incremental |
| `sumMerge(col)` | Merge sum states | Query aggregate table |
| `avgState(col)` | Average state | Store running average |
| `avgMerge(col)` | Merge avg states | Query aggregate table |

---

## 10. Summary: Key Patterns

| Pattern | Implementation | Benefit |
|---------|----------------|---------|
| **ReplacingMergeTree everywhere** | Handle chain reorgs naturally | Automatic deduplication |
| **DateTime64(6)** | Microsecond timestamps | Matches Solana precision |
| **Decimal(38,9) / (76,38)** | Exact financial math | No floating point errors |
| **Array(Tuple(...))** | Nested blockchain data | Efficient storage, typed access |
| **Null engine + MV** | String → typed transformation | Clean ingestion pipeline |
| **arrayExists with lambda** | Filter vote transactions | Separate analytics from noise |
| **AggregatingMergeTree** | Pre-computed metrics | Sub-second dashboard queries |
| **SummingMergeTree** | Simple counters | Automatic merging |
| **generateULID()** | Sortable unique IDs | Better than UUID |
| **IP-based quotas** | Rate limiting | Public API protection |
| **Query cache** | Repeated queries | Faster dashboard loads |
| **EXCHANGE TABLES** | Atomic table swap | Zero-downtime updates |

---

## 11. Additional Patterns (from load_queries.ts)

### 11.1 Queries Metadata Table

```sql
-- Store saved queries with auto-generated IDs
CREATE TABLE default.queries
(
    id String DEFAULT generateULID(),
    name String,
    group String,
    query String,
    chart String DEFAULT '{"type":"line"}',  -- JSON chart config
    format Bool,
    params String DEFAULT '{}'               -- JSON parameters
)
ENGINE = MergeTree()
ORDER BY id
```

### 11.2 Atomic Table Swap Pattern

```sql
-- 1. Create temp table with same schema
CREATE TABLE IF NOT EXISTS default.queries_temp
(
    id String DEFAULT generateULID(),
    name String,
    group String,
    query String,
    chart String DEFAULT '{"type":"line"}',
    format Bool,
    params String DEFAULT '{}'
)
ENGINE = MergeTree()
ORDER BY id;

-- 2. Load data into temp table
INSERT INTO default.queries_temp SELECT * FROM ...;

-- 3. Atomic swap (instant, no downtime)
EXCHANGE TABLES default.queries_temp AND default.queries;

-- 4. Cleanup
DROP TABLE default.queries_temp;
```

**Key Benefit**: Updates are atomic - users always see either old or new data, never partial state.

---

## 12. Complete File Inventory (CryptoHouse)

| File | Checked | Content |
|------|---------|---------|
| `setup.sql` | ✅ | Full schema: 7 core tables, staging tables, MVs, aggregations, users, quotas |
| `queries.json` | ✅ | 32 queries: counts, metrics, OHLC, compute units, sentiment |
| `load_queries.ts` | ✅ | EXCHANGE TABLES pattern, queries metadata table |
| `README.md` | ✅ | Overview, live instance connection |
| `.github/ISSUE_TEMPLATE/*` | ✅ | Templates only, no SQL |
| `package.json` | ✅ | Node.js deps, not relevant |
| `.gitignore`, `.nvmrc`, `LICENSE` | ✅ | Config files, not relevant |

---

# Part 2: StockHouse Real-Time Market Data

**Source**: [ClickHouse/stockhouse](https://github.com/ClickHouse/stockhouse)

**Use Case**: Real-time stock and crypto market data - millions of updates per second via WebSocket

---

## 13. Real-Time Market Data Schema

### 13.1 Stock Quotes Table

```sql
CREATE TABLE quotes
(
    sym LowCardinality(String),          -- Stock symbol (AAPL, MSFT, etc.)
    bx UInt8,                             -- Bid exchange ID
    bp Float64,                           -- Bid price
    bs UInt64,                            -- Bid size
    ax UInt8,                             -- Ask exchange ID
    ap Float64,                           -- Ask price
    as UInt64,                            -- Ask size
    c UInt8,                              -- Condition code
    i Array(UInt8),                       -- Indicators array
    t UInt64,                             -- Timestamp (milliseconds epoch)
    q UInt64,                             -- Quote sequence number
    z UInt8,                              -- Tape
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (sym, t - (t % 60000));          -- Time-bucketed ordering
```

**Key Pattern: Time-Bucketed ORDER BY**
```sql
ORDER BY (sym, t - (t % 60000))
-- Rounds timestamp to 60-second (1 minute) buckets
-- Groups related quotes together on disk
-- Optimizes range queries within time windows
```

### 13.2 Stock Trades Table

```sql
CREATE TABLE trades
(
    sym LowCardinality(String),          -- Stock symbol
    i String,                             -- Trade ID
    x UInt8,                              -- Exchange ID
    p Float64,                            -- Price
    s UInt64,                             -- Size (shares)
    c Array(UInt8),                       -- Condition codes
    t UInt64,                             -- Timestamp (milliseconds epoch)
    q UInt64,                             -- Trade sequence number
    z UInt8,                              -- Tape
    trfi UInt64,                          -- TRF ID
    trft UInt64,                          -- TRF timestamp
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (sym, t - (t % 60000));
```

### 13.3 Crypto Quotes Table

```sql
CREATE TABLE crypto_quotes
(
    pair String,                          -- Trading pair (BTC-USD, ETH-USD)
    bp Float64,                           -- Bid price
    bs Float64,                           -- Bid size
    ap Float64,                           -- Ask price
    as Float64,                           -- Ask size
    t UInt64,                             -- Timestamp (milliseconds)
    x UInt8,                              -- Exchange ID
    r UInt64,                             -- Received timestamp
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (pair, t - (t % 60000));
```

### 13.4 Crypto Trades Table

```sql
CREATE TABLE crypto_trades
(
    pair String,                          -- Trading pair
    p Float64,                            -- Price
    t UInt64,                             -- Timestamp (milliseconds)
    s Float64,                            -- Size
    c Array(UInt8),                       -- Conditions
    i String,                             -- Trade ID
    x UInt8,                              -- Exchange ID
    r UInt64,                             -- Received timestamp
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (pair, t - (t % 60000), i);      -- Include trade ID for uniqueness
```

### 13.5 Fair Market Value Table

```sql
CREATE TABLE stock_fmv
(
    sym String,
    fmv Float64,                          -- Fair market value
    t UInt64,
    inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())
)
ORDER BY (sym, t - (t % 60000));
```

---

## 14. Daily Aggregation Tables (AggregatingMergeTree)

### 14.1 Crypto Trades Daily Aggregation

```sql
CREATE TABLE agg_crypto_trades_daily
(
    event_date Date,
    pair LowCardinality(String),
    open_price_state AggregateFunction(argMin, Float64, UInt64),   -- First price of day
    last_price_state AggregateFunction(argMax, Float64, UInt64),   -- Last price of day
    volume_state AggregateFunction(sum, Float64),                   -- Total volume
    latest_t_state AggregateFunction(max, UInt64)                   -- Latest timestamp
)
PARTITION BY toYYYYMM(event_date)         -- Monthly partitions
ORDER BY (event_date, pair);

CREATE MATERIALIZED VIEW mv_crypto_trades_daily
TO agg_crypto_trades_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    pair,
    argMinState(p, t) AS open_price_state,    -- Price at earliest timestamp
    argMaxState(p, t) AS last_price_state,    -- Price at latest timestamp
    sumState(s) AS volume_state,
    maxState(t) AS latest_t_state
FROM crypto_trades
GROUP BY event_date, pair;
```

**Key Pattern: argMinState/argMaxState for OHLC**
```sql
-- argMinState(value, timestamp) → value at MIN timestamp (OPEN price)
-- argMaxState(value, timestamp) → value at MAX timestamp (CLOSE price)
-- This is the correct way to get open/close prices!

-- Query the aggregated data:
SELECT
    event_date,
    pair,
    argMinMerge(open_price_state) AS open,
    argMaxMerge(last_price_state) AS close,
    sumMerge(volume_state) AS volume
FROM agg_crypto_trades_daily
GROUP BY event_date, pair
```

### 14.2 Crypto Quotes Daily Aggregation

```sql
CREATE TABLE agg_crypto_quotes_daily
(
    event_date Date,
    pair LowCardinality(String),
    bid_state AggregateFunction(argMax, Float64, UInt64),    -- Latest bid
    ask_state AggregateFunction(argMax, Float64, UInt64),    -- Latest ask
    latest_t_state AggregateFunction(max, UInt64)
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, pair);

CREATE MATERIALIZED VIEW mv_crypto_quotes_daily
TO agg_crypto_quotes_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    pair,
    argMaxState(bp, t) AS bid_state,          -- Bid at latest timestamp
    argMaxState(ap, t) AS ask_state,          -- Ask at latest timestamp
    maxState(t) AS latest_t_state
FROM crypto_quotes
GROUP BY event_date, pair;
```

### 14.3 Stock Trades Daily Aggregation

```sql
CREATE TABLE agg_stock_trades_daily
(
    event_date Date,
    sym LowCardinality(String),
    open_price_state AggregateFunction(argMin, Float64, UInt64),
    last_price_state AggregateFunction(argMax, Float64, UInt64),
    volume_state AggregateFunction(sum, UInt64),              -- UInt64 for stock volume
    latest_t_state AggregateFunction(max, UInt64)
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, sym);

CREATE MATERIALIZED VIEW mv_stock_trades_daily
TO agg_stock_trades_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    sym,
    argMinState(p, t) AS open_price_state,
    argMaxState(p, t) AS last_price_state,
    sumState(s) AS volume_state,
    maxState(t) AS latest_t_state
FROM trades
GROUP BY event_date, sym;
```

### 14.4 Stock Quotes Daily Aggregation

```sql
CREATE TABLE agg_stock_quotes_daily
(
    event_date Date,
    sym LowCardinality(String),
    bid_state AggregateFunction(argMax, Float64, UInt64),
    ask_state AggregateFunction(argMax, Float64, UInt64),
    latest_t_state AggregateFunction(max, UInt64)
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, sym);

CREATE MATERIALIZED VIEW mv_stock_quotes_daily
TO agg_stock_quotes_daily
AS SELECT
    toDate(fromUnixTimestamp64Milli(t, 'UTC')) AS event_date,
    sym,
    argMaxState(bp, t) AS bid_state,
    argMaxState(ap, t) AS ask_state,
    maxState(t) AS latest_t_state
FROM quotes
GROUP BY event_date, sym;
```

---

## 15. OHLC Candlestick Query Patterns

### 15.1 Generic OHLC Query Structure

```sql
-- Candlestick with configurable bucket size
SELECT
    toDateTime64(toStartOfInterval(
        fromUnixTimestamp64Milli(t, 'UTC'),
        INTERVAL {bucket_seconds} SECOND
    ), 3) AS timestamp,
    argMin(p, t) AS open,                 -- Price at first timestamp in bucket
    argMax(p, t) AS close,                -- Price at last timestamp in bucket
    max(p) AS high,                       -- Highest price in bucket
    min(p) AS low,                        -- Lowest price in bucket
    sum(s) AS volume                      -- Total volume in bucket
FROM crypto_trades
WHERE pair = {pair:String}
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL {window} MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC
```

### 15.2 Predefined Timeframe Queries

```sql
-- 15-second buckets, 10-minute window (live view)
SELECT
    toDateTime64(toStartOfInterval(fromUnixTimestamp64Milli(t, 'UTC'), INTERVAL 15 SECOND), 3) AS timestamp,
    argMin(p, t) AS open,
    argMax(p, t) AS close,
    max(p) AS high,
    min(p) AS low
FROM crypto_trades
WHERE pair = 'BTC-USD'
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL 10 MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC;

-- 1-minute buckets, 30-minute window
SELECT
    toDateTime64(toStartOfInterval(fromUnixTimestamp64Milli(t, 'UTC'), INTERVAL 1 MINUTE), 3) AS timestamp,
    argMin(p, t) AS open,
    argMax(p, t) AS close,
    max(p) AS high,
    min(p) AS low
FROM crypto_trades
WHERE pair = 'BTC-USD'
    AND t >= toUnixTimestamp64Milli(now64() - INTERVAL 30 MINUTE)
GROUP BY timestamp
ORDER BY timestamp ASC;

-- 2-minute buckets, 1-hour window
-- 30-minute buckets, 24-hour window
-- 10-minute buckets, 24-hour window (for stocks)
```

### 15.3 Live Data Query with Aggregated Tables

```sql
-- Combine daily aggregates with bid/ask spread
WITH trades AS (
    SELECT
        pair,
        argMaxMerge(last_price_state) AS last,
        argMinMerge(open_price_state) AS open,
        sumMerge(volume_state) AS volume
    FROM agg_crypto_trades_daily
    WHERE event_date = toDate(now(), 'UTC')
    GROUP BY pair
),
quotes AS (
    SELECT
        pair,
        argMaxMerge(bid_state) AS bid,
        argMaxMerge(ask_state) AS ask
    FROM agg_crypto_quotes_daily
    WHERE event_date = toDate(now(), 'UTC')
    GROUP BY pair
)
SELECT
    t.pair,
    t.last AS price,
    t.open,
    ((t.last - t.open) / t.open) * 100 AS change_pct,
    t.volume,
    q.bid,
    q.ask,
    q.ask - q.bid AS spread
FROM trades t
JOIN quotes q ON t.pair = q.pair
ORDER BY t.volume DESC;
```

### 15.4 Stock Intraday Query with CTEs

```sql
WITH trades AS (
    SELECT
        sym,
        argMax(p, t) AS last_price,
        argMinIf(p, t, t >= toUnixTimestamp64Milli(toStartOfDay(now(), 'America/New_York'))) AS open_price,
        sum(s) AS volume
    FROM trades
    WHERE toDate(fromUnixTimestamp64Milli(t, 'UTC')) = toDate(now())
    GROUP BY sym
),
quotes AS (
    SELECT
        sym,
        argMax(bp, t) AS bid,
        argMax(ap, t) AS ask
    FROM quotes
    WHERE toDate(fromUnixTimestamp64Milli(t, 'UTC')) = toDate(now())
    GROUP BY sym
)
SELECT
    t.sym,
    t.last_price,
    t.open_price,
    ((t.last_price - t.open_price) / t.open_price) * 100 AS change_pct,
    t.volume,
    q.bid,
    q.ask
FROM trades t
JOIN quotes q ON t.sym = q.sym
ORDER BY volume DESC;
```

---

## 16. High-Throughput Ingestion (Go Ingester)

### 16.1 Batch Insert Pattern

```go
// INSERT statements for each table type
"INSERT INTO %s.quotes (sym,bx,bp,bs,ax,ap,as,c,i,t,q,z) VALUES"
"INSERT INTO %s.trades (sym,i,x,p,s,c,t,q,z,trfi,trft) VALUES"
"INSERT INTO %s.crypto_quotes (pair,bp,bs,ap,as,t,x,r) VALUES"
"INSERT INTO %s.crypto_trades (pair,p,t,s,c,i,x,r) VALUES"
"INSERT INTO %s.stock_fmv (sym,fmv,t) VALUES"
```

### 16.2 Ingester Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| Channel capacity | 200,000 | Buffer for incoming WebSocket messages |
| Max rows per batch | 50,000 | Optimal batch size for inserts |
| Max flush interval | 1 second | Maximum wait before flushing |
| Insert timeout | 10 seconds | Per-batch timeout |
| Backpressure threshold | 95% (190,000) | When to start shedding load |
| Shedding rate | 1,000 items | Drop count when threshold exceeded |

### 16.3 ClickHouse Client Configuration

```go
// Native protocol with LZ4 compression
clickhouse.Open(&clickhouse.Options{
    Addr: []string{host},
    Auth: clickhouse.Auth{
        Database: database,
        Username: username,
        Password: password,
    },
    Compression: &clickhouse.Compression{
        Method: clickhouse.CompressionLZ4,      // LZ4 for speed
    },
    TLS: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
    DialTimeout: 30 * time.Second,
})
```

### 16.4 Key Ingestion Optimizations

| Optimization | Implementation | Benefit |
|--------------|----------------|---------|
| Native protocol | clickhouse-go/v2 | Faster than HTTP |
| LZ4 compression | CompressionLZ4 | 3-4x compression, low CPU |
| Batch inserts | 50,000 rows | Optimal throughput |
| Async buffering | 200k channel | Handle bursts |
| Load shedding | Drop at 95% | Prevent OOM |
| 1-second flush | Max interval | Low latency |

---

## 17. Timestamp Handling

### 17.1 Millisecond Epoch Patterns

```sql
-- Store current time as millisecond epoch
inserted_at UInt64 DEFAULT toUnixTimestamp64Milli(now64())

-- Convert millisecond epoch to DateTime
fromUnixTimestamp64Milli(t, 'UTC') AS datetime

-- Convert DateTime to millisecond epoch
toUnixTimestamp64Milli(now64())

-- Convert to Date for filtering
toDate(fromUnixTimestamp64Milli(t, 'UTC'))

-- Time zone handling
toStartOfDay(now(), 'America/New_York')  -- Market open time
```

### 17.2 Time Bucketing Functions

```sql
-- Round to interval (for ORDER BY optimization)
t - (t % 60000)                          -- Round to minute (60000 ms)
t - (t % 3600000)                        -- Round to hour

-- toStartOfInterval for grouping
toStartOfInterval(datetime, INTERVAL 15 SECOND)
toStartOfInterval(datetime, INTERVAL 1 MINUTE)
toStartOfInterval(datetime, INTERVAL 5 MINUTE)
toStartOfInterval(datetime, INTERVAL 1 HOUR)

-- Convert back to DateTime64 for output
toDateTime64(toStartOfInterval(...), 3)  -- 3 = millisecond precision
```

---

## 18. Query Output Formats

### 18.1 Arrow Format (Default)

```javascript
// Efficient binary format for large datasets
const response = await fetch(url, {
    method: 'POST',
    body: query + ' FORMAT Arrow'
});
```

### 18.2 JSONCompact Format

```javascript
// Human-readable, smaller payload for APIs
const response = await fetch(url, {
    method: 'POST',
    body: query + ' FORMAT JSONCompact'
});
```

---

# Part 3: Operational Patterns (from ClickPy)

---

## 19. Enum8 for Categorical Data

### 19.1 Event Types Pattern

```sql
-- Enum8 uses 1 byte vs LowCardinality variable size
CREATE TABLE events (
    event_type Enum8(
        'trade' = 1,
        'quote' = 2,
        'order' = 3,
        'cancel' = 4,
        'fill' = 5,
        'partial_fill' = 6,
        'reject' = 7,
        'liquidation' = 8
    ),
    -- ... other columns
)
```

### 19.2 Order Status Pattern

```sql
CREATE TABLE orders (
    status Enum8(
        'pending' = 0,
        'open' = 1,
        'filled' = 2,
        'partial' = 3,
        'cancelled' = 4,
        'rejected' = 5,
        'expired' = 6
    ),
    side Enum8(
        'buy' = 1,
        'sell' = 2
    ),
    order_type Enum8(
        'market' = 1,
        'limit' = 2,
        'stop' = 3,
        'stop_limit' = 4,
        'trailing_stop' = 5
    ),
    -- ... other columns
)
```

### 19.3 Action Enum Pattern

```sql
-- From GitHub events - applicable to trading events
action Enum8(
    'none' = 0,
    'created' = 1,
    'added' = 2,
    'edited' = 3,
    'deleted' = 4,
    'opened' = 5,
    'closed' = 6,
    'reopened' = 7,
    'assigned' = 8,
    'unassigned' = 9,
    'labeled' = 10,
    'unlabeled' = 11,
    'synchronize' = 14,
    'started' = 15,
    'published' = 16,
    'merged' = 20
)
```

**Benefits of Enum8:**
- 1 byte storage (vs variable for LowCardinality)
- Compile-time validation of values
- Better compression
- Self-documenting schema

---

## 20. Lightweight Deletes for Data Fixes

### 20.1 Enable Lightweight Delete Mode

```sql
-- Enable lightweight deletes (faster than traditional mutations)
SET lightweight_delete_mode = 'lightweight_update';
```

### 20.2 Delete Specific Date from Aggregate Tables

```sql
-- Delete bad data from all aggregate tables
ALTER TABLE trades_per_day DELETE
WHERE date = CAST('2024-01-15', 'Date');

ALTER TABLE trades_per_day_by_symbol DELETE
WHERE date = CAST('2024-01-15', 'Date');

ALTER TABLE trades_per_day_by_symbol_by_exchange DELETE
WHERE date = CAST('2024-01-15', 'Date');

-- Delete from main table
ALTER TABLE trades DELETE
WHERE toDate(timestamp) = CAST('2024-01-15', 'Date');
```

### 20.3 Full Rebuild Pattern (Drop MV, Rebuild, Exchange)

```sql
-- 1. Drop the materialized view
DROP VIEW IF EXISTS trades_per_month_mv;

-- 2. Create shadow table
CREATE TABLE trades_per_month_v2 AS trades_per_month;

-- 3. Rebuild from source data
INSERT INTO trades_per_month_v2
SELECT
    toStartOfMonth(date) AS month,
    symbol,
    sum(count) AS count,
    sum(volume) AS volume
FROM trades_per_day
GROUP BY month, symbol;

-- 4. Atomic swap
EXCHANGE TABLES trades_per_month_v2 AND trades_per_month;

-- 5. Cleanup
DROP TABLE trades_per_month_v2;

-- 6. Recreate MV
CREATE MATERIALIZED VIEW trades_per_month_mv
TO trades_per_month AS
SELECT
    toStartOfMonth(timestamp) AS month,
    symbol,
    count() AS count,
    sum(quantity) AS volume
FROM trades
GROUP BY month, symbol;
```

---

## 21. Multi-Dimensional Aggregation Pattern

### 21.1 SummingMergeTree for Different Query Patterns

```sql
-- Total by symbol
CREATE TABLE trades_by_symbol (
    symbol LowCardinality(String),
    count Int64,
    volume Decimal(38, 18)
)
ENGINE = SummingMergeTree
ORDER BY symbol;

-- By symbol + date
CREATE TABLE trades_per_day (
    date Date,
    symbol LowCardinality(String),
    count Int64,
    volume Decimal(38, 18)
)
ENGINE = SummingMergeTree
ORDER BY (symbol, date);

-- By symbol + date + exchange
CREATE TABLE trades_per_day_by_exchange (
    date Date,
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    count Int64,
    volume Decimal(38, 18)
)
ENGINE = SummingMergeTree
ORDER BY (symbol, date, exchange);

-- By symbol + date + side
CREATE TABLE trades_per_day_by_side (
    date Date,
    symbol LowCardinality(String),
    side LowCardinality(String),
    count Int64,
    volume Decimal(38, 18)
)
ENGINE = SummingMergeTree
ORDER BY (symbol, date, side);
```

### 21.2 Automatic View Selection

```javascript
// Select smallest view that covers required columns
const views = [
    { name: 'trades_by_symbol', columns: ['symbol'] },
    { name: 'trades_per_day', columns: ['symbol', 'date'] },
    { name: 'trades_per_day_by_exchange', columns: ['symbol', 'date', 'exchange'] },
    { name: 'trades_per_day_by_side', columns: ['symbol', 'date', 'side'] },
];

function selectView(requiredColumns) {
    return views.find(v =>
        requiredColumns.every(c => v.columns.includes(c))
    );
}
```

---

## 22. Complete File Inventory (StockHouse)

| File | Checked | Content |
|------|---------|---------|
| `scripts/schema.sql` | ✅ | 5 raw tables, 4 aggregation tables, 4 MVs |
| `frontend/src/queries/index.js` | ✅ | OHLC queries, live data, time bucketing |
| `ingester-go/ingest.go` | ✅ | Batch settings, LZ4 compression, backpressure |
| `api/query.js` | ✅ | Arrow/JSONCompact format handling |
| `frontend/src/composables/useClickhouse.js` | ✅ | Health check, ping monitoring |
| `README.md` | ✅ | Architecture overview |

---

## 23. Summary: All Trading-Related Patterns

| Pattern | Source | Click Relevance |
|---------|--------|-----------------|
| **Time-bucketed ORDER BY** | StockHouse | High-frequency trade ordering |
| **argMinState/argMaxState** | StockHouse | OHLC open/close prices |
| **AggregateFunction for OHLC** | StockHouse | Daily aggregations |
| **Monthly PARTITION BY** | StockHouse | Time-series partitioning |
| **toUnixTimestamp64Milli** | StockHouse | Millisecond timestamps |
| **fromUnixTimestamp64Milli** | StockHouse | Epoch to DateTime |
| **toStartOfInterval** | StockHouse | Candlestick bucketing |
| **LZ4 compression** | StockHouse | Fast ingestion |
| **50k batch size** | StockHouse | Optimal throughput |
| **Backpressure/load shedding** | StockHouse | Handle bursts |
| **Arrow format** | StockHouse | Efficient data transfer |
| **Null engine staging** | CryptoHouse | String → typed transform |
| **arrayExists lambda** | CryptoHouse | Filter transactions |
| **Decimal(38,9)** | CryptoHouse | Exact financial math |
| **generateULID()** | CryptoHouse | Sortable unique IDs |
| **Enum8** | ClickPy | Order status, event types |
| **lightweight_delete_mode** | ClickPy | Fast data fixes |
| **DROP + rebuild + EXCHANGE** | ClickPy | Atomic aggregate rebuild |
| **Multi-dim SummingMergeTree** | ClickPy | Query-specific aggregates |

---

# Part 4: Data Pipeline Patterns (from sql.clickhouse.com & reversedns.space)

---

## 24. S3Queue Engine for Continuous Ingestion

### 24.1 Basic S3Queue Setup

```sql
-- Continuous ingestion from S3/GCS without external orchestration
CREATE TABLE trades_queue (
    data String
)
ENGINE = S3Queue(
    'https://storage.googleapis.com/bucket/trades/*.parquet',
    'HMAC_KEY',
    'HMAC_SECRET',
    'Parquet'
)
SETTINGS
    mode = 'ordered',                          -- Process files in order
    s3queue_processing_threads_num = 10;       -- Parallel processing
```

### 24.2 S3Queue with Advanced Settings

```sql
CREATE TABLE historical_trades_queue (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    side LowCardinality(String)
)
ENGINE = S3Queue(
    's3://bucket/trades/{YYYY}/{MM}/{DD}/*.parquet',
    'AWS_KEY',
    'AWS_SECRET',
    'Parquet'
)
SETTINGS
    mode = 'unordered',                        -- Process files in any order (faster)
    s3queue_polling_min_timeout_ms = 1800000,  -- 30 min minimum poll interval
    s3queue_polling_max_timeout_ms = 2400000,  -- 40 min maximum poll interval
    s3queue_tracked_files_limit = 2000,        -- Track up to 2000 processed files
    s3queue_buckets = 30;                      -- Parallel processing buckets
```

### 24.3 S3Queue Settings Reference

| Setting | Value | Purpose |
|---------|-------|---------|
| `mode` | 'ordered' | Process files in lexicographic order |
| `mode` | 'unordered' | Process files in any order (faster) |
| `s3queue_polling_min_timeout_ms` | 1800000 | Min wait between polls (30 min) |
| `s3queue_polling_max_timeout_ms` | 2400000 | Max wait between polls (40 min) |
| `s3queue_tracked_files_limit` | 2000 | Max files to track as processed |
| `s3queue_buckets` | 30 | Parallel processing buckets |
| `s3queue_processing_threads_num` | 10 | Parallel processing threads |

### 24.4 Daily Queue Rotation Pattern

```sql
-- Create new queue for today (prevents unbounded file tracking)
CREATE TABLE trades_queue_${today} (...)
ENGINE = S3Queue('s3://bucket/trades/${today}/*.parquet', ...)
SETTINGS mode = 'ordered';

-- Route to main table via MV
CREATE MATERIALIZED VIEW trades_queue_${today}_mv
TO trades AS
SELECT * FROM trades_queue_${today};

-- Drop yesterday's queue (cleanup)
DROP TABLE IF EXISTS trades_queue_${yesterday};
DROP VIEW IF EXISTS trades_queue_${yesterday}_mv;
```

---

## 25. Medallion Architecture (Bronze → Silver → Gold)

### 25.1 Bronze Layer (Raw Ingestion)

```sql
-- Raw trades from WebSocket/Kafka (accepts everything)
CREATE TABLE trades_bronze (
    raw_data String,
    _file LowCardinality(String),
    ingested_at DateTime64(3) DEFAULT now64(3)
)
ENGINE = MergeTree
ORDER BY ingested_at;

-- MV from S3Queue to Bronze
CREATE MATERIALIZED VIEW trades_bronze_mv TO trades_bronze AS
SELECT
    data AS raw_data,
    _file
FROM trades_queue
WHERE isValidJSON(data) = 1;        -- Only valid JSON
```

### 25.2 Silver Layer (Cleaned & Validated)

```sql
-- Parsed and validated trades
CREATE TABLE trades_silver (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    side LowCardinality(String),
    trade_id String,
    dedup_hash UInt64 MATERIALIZED cityHash64(symbol, exchange, trade_id, toString(timestamp))
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, dedup_hash)
PARTITION BY toYYYYMM(timestamp);

-- MV from Bronze to Silver (with validation)
CREATE MATERIALIZED VIEW trades_silver_mv TO trades_silver AS
SELECT
    parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')) AS timestamp,
    JSONExtractString(raw_data, 'symbol') AS symbol,
    JSONExtractString(raw_data, 'exchange') AS exchange,
    JSONExtractFloat(raw_data, 'price')::Decimal(38, 18) AS price,
    JSONExtractFloat(raw_data, 'quantity')::Decimal(38, 18) AS quantity,
    JSONExtractString(raw_data, 'side') AS side,
    JSONExtractString(raw_data, 'trade_id') AS trade_id
FROM trades_bronze
WHERE price > 0 AND quantity > 0;   -- Basic validation
```

### 25.3 Gold Layer (Aggregates)

```sql
-- Pre-aggregated OHLCV for dashboards
CREATE TABLE ohlcv_1m_gold (
    timestamp DateTime,
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    open Decimal(38, 18),
    high Decimal(38, 18),
    low Decimal(38, 18),
    close Decimal(38, 18),
    volume Decimal(38, 18),
    trade_count UInt64
)
ENGINE = SummingMergeTree
ORDER BY (symbol, exchange, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- MV from Silver to Gold
CREATE MATERIALIZED VIEW ohlcv_1m_gold_mv TO ohlcv_1m_gold AS
SELECT
    toStartOfMinute(timestamp) AS timestamp,
    symbol,
    exchange,
    argMin(price, timestamp) AS open,
    max(price) AS high,
    min(price) AS low,
    argMax(price, timestamp) AS close,
    sum(quantity) AS volume,
    count() AS trade_count
FROM trades_silver
GROUP BY timestamp, symbol, exchange;
```

---

## 26. Dead Letter Queue (DLQ) Pattern

### 26.1 DLQ Table

```sql
-- Store records that failed validation
CREATE TABLE trades_dlq (
    raw_data String,
    error_reason LowCardinality(String),
    ingested_at DateTime64(3) DEFAULT now64(3),
    _file LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY ingested_at
TTL ingested_at + INTERVAL 30 DAY;   -- Auto-cleanup after 30 days
```

### 26.2 Route Bad Records to DLQ

```sql
-- Good records → Silver
CREATE MATERIALIZED VIEW trades_silver_mv TO trades_silver AS
SELECT ...
FROM trades_bronze
WHERE price > 0
    AND quantity > 0
    AND isValidJSON(raw_data) = 1
    AND abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) < 86400;

-- Bad records → DLQ (inverted conditions)
CREATE MATERIALIZED VIEW trades_dlq_mv TO trades_dlq AS
SELECT
    raw_data,
    multiIf(
        isValidJSON(raw_data) = 0, 'invalid_json',
        JSONExtractFloat(raw_data, 'price') <= 0, 'invalid_price',
        JSONExtractFloat(raw_data, 'quantity') <= 0, 'invalid_quantity',
        abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) >= 86400, 'stale_timestamp',
        'unknown'
    ) AS error_reason,
    _file
FROM trades_bronze
WHERE price <= 0
    OR quantity <= 0
    OR isValidJSON(raw_data) = 0
    OR abs(dateDiff('second', parseDateTimeBestEffort(JSONExtractString(raw_data, 'timestamp')), now())) >= 86400;
```

---

## 27. WITH FILL for Time Series Gaps

### 27.1 Fill Missing OHLCV Buckets

```sql
-- Ensure no gaps in candlestick data
SELECT
    timestamp,
    symbol,
    open,
    high,
    low,
    close,
    volume
FROM ohlcv_1m
WHERE symbol = 'BTC-USD'
    AND timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp ASC
WITH FILL
    FROM toStartOfMinute(now() - INTERVAL 1 HOUR)
    TO toStartOfMinute(now())
    STEP INTERVAL 1 MINUTE;
```

### 27.2 Fill with Previous Values (Forward Fill)

```sql
SELECT
    timestamp,
    symbol,
    -- Use previous close for missing open
    if(open = 0, lagInFrame(close) OVER (ORDER BY timestamp), open) AS open,
    if(high = 0, lagInFrame(close) OVER (ORDER BY timestamp), high) AS high,
    if(low = 0, lagInFrame(close) OVER (ORDER BY timestamp), low) AS low,
    if(close = 0, lagInFrame(close) OVER (ORDER BY timestamp), close) AS close,
    volume
FROM (
    SELECT *
    FROM ohlcv_1m
    WHERE symbol = 'BTC-USD'
    ORDER BY timestamp
    WITH FILL
        FROM toStartOfMinute(now() - INTERVAL 1 HOUR)
        TO toStartOfMinute(now())
        STEP INTERVAL 1 MINUTE
)
```

### 27.3 Fill Grid for Heatmaps

```sql
-- Complete hour x day grid
SELECT
    hour,
    day_of_week,
    count
FROM trades_by_hour_dow
ORDER BY hour, day_of_week
WITH FILL
    FROM 0 TO 24         -- Hours 0-23
    STEP 1
WITH FILL
    FROM 1 TO 8          -- Days 1-7
    STEP 1;
```

---

## 28. JSON Handling Patterns

### 28.1 JSON SKIP Clause (Skip Large Nested Fields)

```sql
-- Skip large/unused nested fields to save storage
CREATE TABLE orders_raw (
    data JSON(
        SKIP `nested.large_field`,           -- Skip specific paths
        SKIP `audit.full_history`,
        SKIP `metadata.raw_response`
    ),
    order_id String MATERIALIZED JSONExtractString(data, 'order_id'),
    symbol LowCardinality(String) MATERIALIZED JSONExtractString(data, 'symbol'),
    timestamp DateTime64(3) MATERIALIZED parseDateTimeBestEffort(JSONExtractString(data, 'timestamp'))
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, order_id);
```

### 28.2 isValidJSON() for Filtering

```sql
-- Filter invalid JSON before processing
CREATE MATERIALIZED VIEW trades_mv TO trades AS
SELECT *
FROM trades_queue
WHERE isValidJSON(data) = 1;

-- Query to find invalid records
SELECT count() AS invalid_count
FROM trades_queue
WHERE isValidJSON(data) = 0;
```

### 28.3 MATERIALIZED Columns from JSON

```sql
CREATE TABLE events (
    raw_json String,
    -- Computed once at insert time, not query time
    event_type LowCardinality(String) MATERIALIZED JSONExtractString(raw_json, 'type'),
    timestamp DateTime64(3) MATERIALIZED parseDateTimeBestEffort(JSONExtractString(raw_json, 'ts')),
    symbol LowCardinality(String) MATERIALIZED JSONExtractString(raw_json, 'symbol'),
    dedup_hash UInt64 MATERIALIZED cityHash64(raw_json)
)
ENGINE = ReplacingMergeTree
ORDER BY (event_type, timestamp, dedup_hash);
```

---

## 29. Deduplication with cityHash64

### 29.1 Hash-Based Deduplication

```sql
-- Create dedup hash from key fields
CREATE TABLE trades (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    exchange LowCardinality(String),
    trade_id String,
    price Decimal(38, 18),
    quantity Decimal(38, 18),
    -- Computed dedup hash
    dedup_hash UInt64 MATERIALIZED cityHash64(
        symbol,
        exchange,
        trade_id,
        toString(timestamp)
    )
)
ENGINE = ReplacingMergeTree
ORDER BY (symbol, timestamp, dedup_hash);
```

### 29.2 Query with Deduplication

```sql
-- Use FINAL to get deduplicated results
SELECT *
FROM trades FINAL
WHERE symbol = 'BTC-USD'
    AND timestamp >= now() - INTERVAL 1 HOUR;

-- Or use subquery for better performance on large tables
SELECT *
FROM (
    SELECT *,
        row_number() OVER (PARTITION BY dedup_hash ORDER BY timestamp DESC) AS rn
    FROM trades
    WHERE symbol = 'BTC-USD'
)
WHERE rn = 1;
```

---

## 30. TTL and Performance Settings

### 30.1 ttl_only_drop_parts (Faster TTL Cleanup)

```sql
-- Drop entire parts instead of row-by-row deletion
CREATE TABLE tick_data (
    timestamp DateTime64(3),
    symbol LowCardinality(String),
    price Decimal(38, 18),
    quantity Decimal(38, 18)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (symbol, timestamp)
TTL timestamp + INTERVAL 7 DAY
SETTINGS ttl_only_drop_parts = 1;      -- Key setting!
```

### 30.2 force_primary_key (Prevent Full Scans)

```sql
-- Require queries to use primary key
SET force_primary_key = 1;

-- This will fail (no primary key filter):
SELECT * FROM trades WHERE price > 100;  -- Error!

-- This will work:
SELECT * FROM trades WHERE symbol = 'BTC-USD' AND price > 100;  -- OK
```

### 30.3 do_not_merge_across_partitions_select_final

```sql
-- Optimize FINAL queries on partitioned tables
SET do_not_merge_across_partitions_select_final = 1;

-- Now FINAL only deduplicates within each partition (much faster)
SELECT *
FROM trades FINAL
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-02-01';
```

---

## 31. Updated Summary: All Patterns

| Pattern | Source | Click Relevance |
|---------|--------|-----------------|
| **S3Queue engine** | sql.clickhouse.com | Ingest historical trades from S3/GCS |
| **S3Queue settings** | sql.clickhouse.com | Polling, tracking, parallelism |
| **Daily queue rotation** | sql.clickhouse.com | Manage unbounded file lists |
| **Medallion architecture** | Bluesky | Bronze → Silver → Gold pipeline |
| **DLQ pattern** | Bluesky | Handle bad trades/quotes |
| **isValidJSON()** | Bluesky | Filter malformed data |
| **JSON SKIP clause** | Bluesky | Skip large nested fields |
| **MATERIALIZED columns** | Bluesky | Compute at insert time |
| **cityHash64 dedup** | Bluesky | Hash-based deduplication |
| **WITH FILL** | reversedns.space | Fill OHLCV time gaps |
| **force_primary_key** | reversedns.space | Prevent full scans |
| **ttl_only_drop_parts** | Bluesky | Faster TTL cleanup |
| **do_not_merge_across_partitions_select_final** | Bluesky | Faster FINAL queries |
| **parseDateTimeBestEffort** | sql.clickhouse.com | Flexible timestamp parsing |
| **multiIf for error routing** | Bluesky | Categorize DLQ errors |
| **lagInFrame for forward fill** | sql.clickhouse.com | Fill missing values |

---

## 32. Additional Query Patterns (from CryptoHouse queries.json)

### 32.1 Conditional Aggregates

```sql
-- maxIf: Aggregate only when condition is true
SELECT
    signature AS id,
    maxIf(
        toInt64OrNull(splitByString(' ', log)[4]),
        log LIKE 'Program % consumed % of % compute units'
    ) AS computed_units_spent
FROM solana.transactions
ARRAY JOIN log_messages AS log
WHERE block_slot = 293312554
GROUP BY id
```

### 32.2 Safe Type Casting

```sql
-- toInt64OrNull: Returns NULL instead of error on invalid input
toInt64OrNull(splitByString(' ', log)[4])

-- ::Type casting syntax (PostgreSQL-style)
JSONExtractFloat(raw_data, 'price')::Decimal(38, 18)
'2024-11-05'::Date32
maker_asset_id::text
```

### 32.3 Custom Time Bucketing

```sql
-- FLOOR for custom bucket sizes (10-second buckets)
FLOOR(toUnixTimestamp(timestamp) / 10) * 10 AS interval_start

-- toStartOfInterval for standard buckets
toStartOfInterval(timestamp, INTERVAL 10 SECOND) AS time_interval
```

### 32.4 any() and anyLast() for OHLC

```sql
-- any(): First value encountered (OPEN price)
-- anyLast(): Last value encountered (CLOSE price)
SELECT
    time_interval,
    any(filled_price) AS open_price,           -- First value
    anyLast(filled_price) AS close_price,      -- Last value
    MIN(filled_price) AS low_price,
    MAX(filled_price) AS high_price
FROM filled_prices
GROUP BY time_interval
```

### 32.5 anyLast() OVER for Forward Fill

```sql
-- Fill zero/missing values with previous non-zero value
SELECT
    time_interval,
    price,
    IF(price = 0,
       anyLast(price) OVER (ORDER BY time_interval),
       price
    ) AS filled_price
FROM prices
```

### 32.6 Parameterized Queries

```sql
-- ClickHouse parameterized query syntax
SELECT uniqCombinedMerge(14)(trx_count) AS unique_transactions
FROM solana.tx_signature_by_day_per_program
WHERE program_id = {program_id:String}    -- Parameter with type

-- Common parameter types:
-- {param:String}
-- {param:UInt64}
-- {param:Date}
-- {param:DateTime}
```

### 32.7 Interval Functions

```sql
-- toIntervalWeek, toIntervalDay, toIntervalMonth
WHERE day >= (today() - toIntervalWeek(1))
WHERE day > (today() - toIntervalMonth(6))
WHERE t >= toUnixTimestamp64Milli(now64() - INTERVAL 10 MINUTE)

-- now() - INTERVAL syntax
now() - INTERVAL 1 WEEK
now() - INTERVAL 1 HOUR
now64() - INTERVAL 30 MINUTE
```

### 32.8 Rolling Window Aggregates

```sql
-- 7-day rolling average
SELECT
    toStartOfDay(block_timestamp) AS day,
    count() AS txns,
    ROUND(AVG(txns) OVER (
        ORDER BY day ASC
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    )) AS `7d_avg`
FROM base.transactions
GROUP BY day
```

### 32.9 Cumulative Sum

```sql
-- Running total with window function
SELECT
    toStartOfDay(min_date) AS day,
    count() AS new_users,
    sum(new_users) OVER (ORDER BY day ASC) AS total_addresses
FROM ethereum.first_time_for_address
GROUP BY day
```

### 32.10 numbers() Table Function

```sql
-- Generate sequence for testing or filling
SELECT count() FROM numbers(10000000000)  -- 10 billion rows

-- Generate date range
SELECT toDate('2024-01-01') + number AS date
FROM numbers(365)
```

---

## 33. Complete Coverage Summary

### CryptoHouse Files Checked

| File | Status | Content |
|------|--------|---------|
| `setup.sql` | ✅ | Solana schemas (7 tables), staging, MVs, users |
| `queries.json` | ✅ | 32 queries with all patterns extracted |
| `load_queries.ts` | ✅ | EXCHANGE TABLES, queries metadata |
| `README.md` | ✅ | Connection info |
| `.github/ISSUE_TEMPLATE/*` | ✅ | Templates only |

**Note**: Production CryptoHouse has additional tables (instructions, polymarket.*, ethereum.*, base.*) not in public repo but query patterns are documented.

### StockHouse Files Checked

| File | Status | Content |
|------|--------|---------|
| `scripts/schema.sql` | ✅ | 5 raw tables, 4 agg tables, 4 MVs |
| `frontend/src/queries/index.js` | ✅ | All OHLC query patterns |
| `ingester-go/ingest.go` | ✅ | Batch settings, LZ4, backpressure |
| `api/query.js` | ✅ | Arrow/JSONCompact formats |
| `frontend/src/api/clickhouse.js` | ✅ | Client config (no SQL) |
| `frontend/src/composables/*.js` | ✅ | Query references (no new SQL) |
| `frontend/src/utils/index.js` | ✅ | Utilities only |
| `server.js` | ✅ | Proxy config, format handling |

### Total Patterns Documented

| Category | Count |
|----------|-------|
| Schema patterns | 15+ |
| Aggregation patterns | 12+ |
| Query patterns | 20+ |
| Ingestion patterns | 8+ |
| Security patterns | 6+ |
| Performance settings | 8+ |
| **Total** | **70+** |
