# Part 8: Risk Analysis, Wallet Funding & External Data Sources

*Building comprehensive token and wallet risk scoring with on-chain analysis, CEX attribution, and external security APIs*

---

## Introduction

The Solana memecoin ecosystem is a high-risk environment where rug pulls, honeypots, and coordinated insider trading are common. Building a trustworthy trading platform requires comprehensive risk analysis that goes beyond simple price metrics.

In this part, we explore the multi-layered risk analysis system that powers token discovery and wallet classification:

1. **Wallet Funding Source Tracking** - Identify where traders' funds originated (CEX attribution, sybil detection)
2. **Token Meta Classification** - AI-powered categorization of tokens by narrative/theme
3. **External Security APIs** - Integration with GoPlus, RugCheck, and Solscan for security intelligence
4. **Risk Score Methodology** - Composite risk calculations for rug probability assessment

---

## The Risk Analysis Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          RISK ANALYSIS ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                         ON-CHAIN DATA                                     │  │
│   │                                                                           │  │
│   │   • Token holders & distribution                                         │  │
│   │   • LP lock/burn status                                                  │  │
│   │   • Dev wallet holdings                                                  │  │
│   │   • Sniper/bundler behavior                                              │  │
│   │   • Mint/freeze authority status                                         │  │
│   └───────────────────────────────────┬──────────────────────────────────────┘  │
│                                       │                                         │
│   ┌──────────────────────────────────┐│┌─────────────────────────────────────┐  │
│   │    EXTERNAL SECURITY APIS        │││     INTERNAL KNOWLEDGE BASE        │  │
│   │                                  │││                                      │  │
│   │   GoPlus Security:               │││   CEX Wallets: 507 addresses        │  │
│   │   • Token security flags         │││   MEV Tips: 72 addresses            │  │
│   │   • Honeypot detection           │││   Platform Fees: 51 addresses       │  │
│   │   • Wallet blacklists            │││   Known Rug Deployers: dynamic      │  │
│   │                                  │││                                      │  │
│   │   RugCheck:                      │││   AI Classification:                │  │
│   │   • Risk scores                  │││   • Token narrative analysis        │  │
│   │   • Deployer flags               │││   • Social media signals            │  │
│   │                                  │││   • Category assignment             │  │
│   │   Solscan Pro:                   │││                                      │  │
│   │   • Transaction history          │││                                      │  │
│   │   • Dev funding source           │││                                      │  │
│   └──────────────────────────────────┘│└─────────────────────────────────────┘  │
│                                       │                                         │
│                                       ▼                                         │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                      COMPOSITE RISK SCORING                              │  │
│   │                                                                           │  │
│   │   rug_risk_pct = dev_holdings(35%) + insider(25%)                        │  │
│   │                + liquidity_risk(25%) + contract_risk(15%)                │  │
│   │                                                                           │  │
│   │   ┌─────────────────────────────────────────────────────────────────┐   │  │
│   │   │  0-20%    │  20-40%    │  40-60%   │  60-80%   │  80-100%      │   │  │
│   │   │  MINIMAL  │  LOW       │  MEDIUM   │  HIGH     │  CRITICAL     │   │  │
│   │   │  Green    │  Yellow    │  Orange   │  Red      │  Dark Red     │   │  │
│   │   └─────────────────────────────────────────────────────────────────┘   │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                       │                                         │
│                                       ▼                                         │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                         CLICKHOUSE STORAGE                               │  │
│   │                                                                           │  │
│   │   wallet_funding_source    token_meta_classification    token_metrics   │  │
│   │   (ReplacingMergeTree)     (ReplacingMergeTree)         (with risk)     │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Wallet Funding Source Tracking

### The Problem: Who Funded This Wallet?

When a new token launches and attracts trading, key questions arise:
- **Is the developer a fresh wallet funded from Binance?** (Common rug pattern)
- **Are multiple "different" traders actually sybils funded from the same source?**
- **Is this whale actually an institutional player from Coinbase?**

### CEX Attribution Database

We maintain a database of **507 known exchange wallets** for attribution:

| Exchange | Wallet Count | Notes |
|----------|--------------|-------|
| Binance | 50+ | Hot wallets, cold wallets |
| Coinbase | 30+ | Including Coinbase Prime |
| OKX | 40+ | Multiple hot wallets |
| Kraken | 20+ | |
| Bybit | 25+ | |
| KuCoin | 20+ | |
| Backpack | 10+ | Solana-native exchange |
| Gate.io | 15+ | |
| MEXC | 20+ | |
| Others | 270+ | 40+ smaller exchanges |

### Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WALLET FUNDING SOURCE PIPELINE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐                                                          │
│   │ New Wallet  │  Trader wallet seen in trade events                      │
│   │ Detected    │                                                          │
│   └──────┬──────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    DataStreams Transform                             │  │
│   │                                                                       │  │
│   │   1. Check state cache for existing funding source                   │  │
│   │      └── If found: skip (already processed)                          │  │
│   │                                                                       │  │
│   │   2. Query Solscan Pro API for first SOL transfer                   │  │
│   │      └── GET /account/transfer?account={wallet}&token=SOL           │  │
│   │                                                                       │  │
│   │   3. Find the first incoming SOL transfer                            │  │
│   │      └── Sort by block_time ascending, take first                    │  │
│   │                                                                       │  │
│   │   4. Match source wallet against CEX_WALLETS constant                │  │
│   │      └── If match: funded_by = "Binance" (or other CEX)             │  │
│   │      └── If no match: funded_by = null (unknown)                     │  │
│   │                                                                       │  │
│   │   5. Cache result in state + emit to NATS                            │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                          │
│   │ NATS:       │  solana.wallet_funding_source                           │
│   │ RowBinary   │                                                          │
│   └──────┬──────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        ClickHouse                                    │  │
│   │                                                                       │  │
│   │   wallet_funding_source (ReplacingMergeTree)                        │  │
│   │   └── One row per wallet, immutable once determined                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Struct Definition

```rust
/// Funding source for a wallet - tracks where the wallet's first SOL came from
#[derive(Debug, Clone, Serialize, Deserialize, Row, AnyState)]
pub struct WalletFundingSource {
    /// The wallet address being analyzed
    pub wallet: String,

    /// Name of the funding source (CEX name or None)
    /// Examples: "Binance", "OKX", "Coinbase", None
    pub funded_by: Option<String>,

    /// The wallet address that provided the funding
    pub source_wallet: String,

    /// Timestamp of the funding transfer
    pub timestamp: u64,

    /// Amount of SOL transferred
    pub amount_sol: f64,
}
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- WALLET FUNDING SOURCE TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: Track where wallets received their initial SOL funding
-- Engine: ReplacingMergeTree - One row per wallet (immutable)
-- Use Cases: Sybil detection, dev funding analysis, trader segmentation
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.wallet_funding_source ON CLUSTER `server-apex`
(
    -- Wallet being analyzed
    wallet String COMMENT 'Wallet address being analyzed',

    -- Funding source info
    funded_by Nullable(String) COMMENT 'CEX name (Binance, OKX, etc.) or NULL if unknown',
    source_wallet String COMMENT 'Wallet address that provided initial funding',

    -- Transfer details
    timestamp UInt64 COMMENT 'Unix timestamp of funding transfer',
    amount_sol Float64 COMMENT 'Amount of SOL transferred',

    -- Metadata
    last_checked DateTime64(9) DEFAULT now64(9) COMMENT 'When this was verified',

    -- Version for deduplication (funding doesn't change, so use timestamp)
    version UInt64 MATERIALIZED timestamp,

    -- ═══════════════════════════════════════════════════════════════════════
    -- COMPUTED FIELDS
    -- ═══════════════════════════════════════════════════════════════════════

    -- Is this wallet funded from a known CEX?
    is_cex_funded Bool MATERIALIZED funded_by IS NOT NULL
        COMMENT 'True if funded from known CEX',

    -- Funding date for time-based analysis
    funding_date Date MATERIALIZED toDate(fromUnixTimestamp(timestamp))
        COMMENT 'Date of funding transfer',

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES
    -- ═══════════════════════════════════════════════════════════════════════

    INDEX idx_funded_by funded_by TYPE set(100) GRANULARITY 1,
    INDEX idx_source source_wallet TYPE bloom_filter GRANULARITY 4,
    INDEX idx_cex is_cex_funded TYPE set(2) GRANULARITY 1
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/wallet_funding_source',
    '{replica}',
    version
)
ORDER BY (wallet)
COMMENT 'Wallet funding source attribution. Immutable once determined.'
SETTINGS index_granularity = 8192;
```

### Analytics: Sybil Detection

```sql
-- Find potential sybil clusters: multiple wallets funded from same source
SELECT
    source_wallet,
    funded_by,
    count() AS wallet_count,
    groupArray(wallet) AS funded_wallets,
    sum(amount_sol) AS total_funded_sol,
    min(fromUnixTimestamp(timestamp)) AS first_funding,
    max(fromUnixTimestamp(timestamp)) AS last_funding
FROM datastreams.wallet_funding_source FINAL
WHERE source_wallet != ''
GROUP BY source_wallet, funded_by
HAVING wallet_count >= 5  -- Suspicious: one wallet funding 5+ traders
ORDER BY wallet_count DESC
LIMIT 50;
```

### Analytics: Dev Funding Analysis

```sql
-- Join funding source with discovery tokens to analyze dev wallet funding
SELECT
    d.token_mint,
    d.token_symbol,
    d.creator AS dev_wallet,
    f.funded_by AS dev_funded_by,
    f.source_wallet AS dev_funding_address,
    f.amount_sol AS initial_funding_sol,
    fromUnixTimestamp(f.timestamp) AS funding_date,

    -- Token metrics
    d.market_cap_usd,
    d.volume_24h_usd,

    -- Risk indicators
    d.snipers_pct,
    d.bundler_pct,
    d.dev_holdings_pct,

    -- Risk flag: fresh CEX wallet is common rug pattern
    if(f.is_cex_funded AND dateDiff('day', f.funding_date, d.created_at) < 7,
       'FRESH_CEX_DEV', 'NORMAL') AS dev_risk_flag

FROM datastreams.discovery FINAL AS d
LEFT JOIN datastreams.wallet_funding_source FINAL AS f
    ON d.creator = f.wallet
WHERE d.stage = 'newly_created'
ORDER BY d.created_at DESC
LIMIT 100;
```

### Analytics: Trader Segmentation

```sql
-- Segment traders by funding source for behavioral analysis
SELECT
    CASE
        WHEN f.funded_by IN ('Binance', 'Coinbase', 'OKX', 'Kraken') THEN 'Major CEX'
        WHEN f.funded_by IS NOT NULL THEN 'Other CEX'
        ELSE 'DeFi Native'
    END AS trader_segment,

    count(DISTINCT p.trader) AS unique_traders,
    count() AS total_positions,
    sum(p.total_spent_usd) AS total_volume_usd,
    avg(p.realized_pnl_usd) AS avg_pnl_usd,
    avg(p.total_spent_usd) AS avg_position_size

FROM datastreams.position_overview FINAL AS p
LEFT JOIN datastreams.wallet_funding_source FINAL AS f
    ON p.trader = f.wallet
GROUP BY trader_segment
ORDER BY unique_traders DESC;
```

---

## 2. Token Meta Classification

### The Problem: What Type of Token Is This?

The Solana token ecosystem has distinct "metas" or narratives that drive trading behavior. Understanding a token's category helps with:
- **Trending Page Organization**: Group tokens by theme (AI, Memes, Gaming)
- **Risk Assessment**: Memecoins have different risk profiles than DeFi tokens
- **Trader Targeting**: Some traders specialize in specific categories

### Classification Categories

| Category | Description | Examples |
|----------|-------------|----------|
| `AI` | AI/ML related tokens | AI agents, ML platforms |
| `Gaming` | Gaming and metaverse | Play-to-earn, virtual worlds |
| `Memes` | Meme coins | Dog coins, cat coins, pepe variants |
| `DeFi` | Decentralized finance | DEX tokens, lending, yield |
| `Infrastructure` | Base layer / tooling | Bridges, oracles, indexers |
| `Social` | Social-Fi tokens | Creator tokens, social platforms |
| `NFT` | NFT-related tokens | NFT marketplaces, collections |
| `Other` | Uncategorized | Fallback category |

### AI Classification Process

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       TOKEN CLASSIFICATION PIPELINE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      INPUT SIGNALS                                   │  │
│   │                                                                       │  │
│   │   Token Metadata:                                                    │  │
│   │   • Name: "Doge AI Agent"                                           │  │
│   │   • Symbol: "DOGEAI"                                                │  │
│   │   • Description: "The first AI-powered dog coin..."                │  │
│   │                                                                       │  │
│   │   Social Signals:                                                    │  │
│   │   • Twitter bio and tweets                                          │  │
│   │   • Telegram group description                                       │  │
│   │   • Website content                                                  │  │
│   │                                                                       │  │
│   │   Trading Patterns:                                                  │  │
│   │   • Holder behavior                                                  │  │
│   │   • Volume patterns                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      AI CLASSIFICATION                               │  │
│   │                                                                       │  │
│   │   Model analyzes signals and returns:                               │  │
│   │   • Primary category: "AI"                                          │  │
│   │   • Sub-category: "AI Agents"                                       │  │
│   │   • Confidence: 0.85                                                │  │
│   │   • Keywords: ["ai", "agent", "doge", "meme"]                       │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      CLICKHOUSE STORAGE                              │  │
│   │                                                                       │  │
│   │   token_meta_classification (ReplacingMergeTree)                    │  │
│   │   • One row per token                                                │  │
│   │   • Updated when token data changes                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse Implementation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- TOKEN META CLASSIFICATION TABLE
-- ═══════════════════════════════════════════════════════════════════════════
-- Purpose: AI-generated token categorizations for trending page
-- Engine: ReplacingMergeTree - Keeps latest classification per token
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE datastreams.token_meta_classification ON CLUSTER `server-apex`
(
    -- ═══════════════════════════════════════════════════════════════════════
    -- TOKEN IDENTIFICATION
    -- ═══════════════════════════════════════════════════════════════════════

    token_mint String COMMENT 'Token mint address (primary key)',
    token_symbol String COMMENT 'Token symbol for display',
    token_name String COMMENT 'Token name for display',

    -- ═══════════════════════════════════════════════════════════════════════
    -- CLASSIFICATION RESULTS
    -- ═══════════════════════════════════════════════════════════════════════

    meta_category LowCardinality(String)
        COMMENT 'Primary category: AI, Gaming, Memes, DeFi, Infrastructure, Social, NFT, Other',

    sub_category Nullable(String)
        COMMENT 'Optional sub-category for finer grouping (e.g., AI Agents, Dog Coins)',

    confidence_score Decimal(5, 4)
        COMMENT 'AI confidence score 0.0000 to 1.0000',

    -- ═══════════════════════════════════════════════════════════════════════
    -- CLASSIFICATION METADATA
    -- ═══════════════════════════════════════════════════════════════════════

    classification_source LowCardinality(String) DEFAULT 'ai'
        COMMENT 'Source: ai, manual, rule_based',

    keywords Array(String) DEFAULT []
        COMMENT 'Keywords that triggered this classification',

    -- ═══════════════════════════════════════════════════════════════════════
    -- TIMESTAMPS
    -- ═══════════════════════════════════════════════════════════════════════

    classified_at DateTime64(3) DEFAULT now64(3)
        COMMENT 'When classification was made',

    version UInt64 MATERIALIZED toUnixTimestamp64Milli(classified_at),

    -- ═══════════════════════════════════════════════════════════════════════
    -- INDEXES
    -- ═══════════════════════════════════════════════════════════════════════

    INDEX idx_category meta_category TYPE set(20) GRANULARITY 1,
    INDEX idx_confidence confidence_score TYPE minmax GRANULARITY 4
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/token_meta_classification',
    '{replica}',
    version
)
ORDER BY token_mint
COMMENT 'AI-classified token categories for Trending Page meta clustering.'
SETTINGS index_granularity = 8192;
```

### Trending Page Integration

```sql
-- Trending tokens by meta category with volume ranking
SELECT
    m.meta_category,
    t.token_mint,
    t.token_symbol,
    t.price_usd,
    t.stats_1h_price_change_pct AS change_1h,
    t.stats_24h_volume_usd AS volume_24h,
    t.market_cap_usd,
    m.confidence_score,

    -- Rank within category by volume
    row_number() OVER (PARTITION BY m.meta_category ORDER BY t.stats_24h_volume_usd DESC) AS category_rank

FROM datastreams.token_metrics FINAL AS t
INNER JOIN datastreams.token_meta_classification FINAL AS m
    ON t.token_mint = m.token_mint
WHERE m.confidence_score >= 0.7  -- High-confidence classifications only
  AND t.stats_24h_volume_usd >= 10000  -- Minimum volume filter
ORDER BY m.meta_category, category_rank
LIMIT 100;
```

```sql
-- Meta category leaderboard (which narrative is hottest?)
SELECT
    m.meta_category,
    count() AS token_count,
    sum(t.stats_24h_volume_usd) AS total_volume_24h,
    avg(t.stats_1h_price_change_pct) AS avg_change_1h,
    max(t.stats_1h_price_change_pct) AS top_gainer_1h,

    -- Category momentum
    sum(t.stats_24h_volume_usd) / nullif(sum(t.stats_6h_volume_usd) * 4, 0) AS volume_acceleration

FROM datastreams.token_metrics FINAL AS t
INNER JOIN datastreams.token_meta_classification FINAL AS m
    ON t.token_mint = m.token_mint
WHERE m.confidence_score >= 0.5
GROUP BY m.meta_category
ORDER BY total_volume_24h DESC;
```

---

## 3. External Security APIs

### Overview of Data Sources

| Source | Data Provided | Rate Limit | Cost |
|--------|---------------|------------|------|
| **GoPlus** | Token security, wallet blacklist, honeypot detection | 100/day free, 10K/day pro | Free/Paid |
| **RugCheck** | Token risk score, deployer flags, rug history | Unlimited | Free |
| **Solscan Pro** | Transaction history, dev funding source | 100K/month | Paid |

### GoPlus Security API

GoPlus provides comprehensive security intelligence for tokens and wallets.

**Token Security Endpoint:**
```
GET https://api.gopluslabs.io/api/v1/solana/token_security/{token_address}
```

**Key Response Fields:**

| Field | Type | Meaning |
|-------|------|---------|
| `is_mintable` | "0"/"1" | Can creator mint more tokens? |
| `is_honeypot` | "0"/"1" | Is this a honeypot trap? |
| `transfer_pausable` | "0"/"1" | Can transfers be frozen? |
| `is_blacklisted` | "0"/"1" | Is token on known blacklist? |
| `creator_percent` | "0.05" | Creator's % of supply |
| `holder_count` | "1234" | Number of holders |

**Wallet Security Endpoint:**
```
GET https://api.gopluslabs.io/api/v1/address_security/{address}
```

**Key Risk Flags:**

| Flag | Meaning |
|------|---------|
| `phishing_activities` | Wallet flagged for phishing |
| `blacklist_doubt` | On suspicious wallet list |
| `sanctioned` | OFAC or similar sanctions |
| `honeypot_related_address` | Connected to honeypot tokens |
| `number_of_malicious_contracts_created` | Creator of scam contracts |

### RugCheck API

RugCheck provides community-driven risk assessment.

**Token Report Endpoint:**
```
GET https://api.rugcheck.xyz/v1/tokens/{token_address}/report
```

**Response Structure:**

```json
{
  "mint": "...",
  "score": 65,          // 0-100, higher = safer
  "rugged": false,      // Token has rugged
  "risks": [
    {
      "name": "Freeze Authority Enabled",
      "description": "Token owner can freeze accounts",
      "level": "danger",   // danger, warning, info
      "score": 25          // Points deducted
    }
  ],
  "topHolders": [...]
}
```

**Risk Levels:**
- `danger`: High risk (20-50 points deducted)
- `warning`: Medium risk (5-20 points deducted)
- `info`: Low risk (0-5 points deducted)

### Caching Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CACHING ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    REDIS (Hot Cache)                                 │  │
│   │                                                                       │  │
│   │   Token Security:                                                    │  │
│   │   • Key: goplus:token:{token_address}                               │  │
│   │   • TTL: 1 hour (security can change)                               │  │
│   │   • Refresh: On miss or TTL expiry                                  │  │
│   │                                                                       │  │
│   │   Wallet Security:                                                   │  │
│   │   • Key: goplus:wallet:{wallet_address}                             │  │
│   │   • TTL: 24 hours (wallet status rarely changes)                    │  │
│   │   • Refresh: On miss or TTL expiry                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    CLICKHOUSE (Permanent)                            │  │
│   │                                                                       │  │
│   │   Dev Funding Cache:                                                 │  │
│   │   • Table: wallet_funding_source                                    │  │
│   │   • TTL: Permanent (historical data never changes)                  │  │
│   │   • Refresh: Never (first SOL transfer is immutable)                │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ClickHouse: External Security Data Cache

```sql
-- Store GoPlus token security data for offline analysis
CREATE TABLE datastreams.token_security_cache ON CLUSTER `server-apex`
(
    -- Token identifier
    token_mint String COMMENT 'Token mint address',

    -- GoPlus security flags
    is_mintable Bool COMMENT 'Can mint more tokens',
    is_honeypot Bool COMMENT 'Honeypot detected',
    transfer_pausable Bool COMMENT 'Can freeze transfers',
    is_blacklisted Bool COMMENT 'On GoPlus blacklist',

    -- Holder info
    creator_address String COMMENT 'Token creator wallet',
    creator_percent Decimal(10, 6) COMMENT 'Creator holding %',
    holder_count UInt32 COMMENT 'Number of holders',

    -- Tax info
    buy_tax Decimal(5, 2) COMMENT 'Buy tax %',
    sell_tax Decimal(5, 2) COMMENT 'Sell tax %',

    -- Metadata
    fetched_at DateTime64(3) DEFAULT now64(3),
    source LowCardinality(String) DEFAULT 'goplus',
    version UInt64 MATERIALIZED toUnixTimestamp64Milli(fetched_at)
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/token_security_cache',
    '{replica}',
    version
)
ORDER BY token_mint
TTL fetched_at + INTERVAL 24 HOUR  -- Re-fetch daily
COMMENT 'Cached GoPlus token security data. TTL: 24 hours.';
```

---

## 4. Risk Score Methodology

### Composite Rug Risk Score

The `rug_risk_pct` is a weighted composite of four risk components:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       RUG RISK SCORE COMPONENTS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   rug_risk_pct = (dev_holdings × 0.35)                                     │
│                + (insider_pct × 0.25)                                       │
│                + (liquidity_risk × 0.25)                                    │
│                + (contract_risk × 0.15)                                     │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                       │  │
│   │   dev_holdings_pct (35% weight)                                      │  │
│   │   └── % of supply held by deployer/creator wallet                    │  │
│   │   └── Source: On-chain token accounts                                │  │
│   │                                                                       │  │
│   │   insider_pct (25% weight)                                           │  │
│   │   └── top10_holdings_pct + snipers_pct                               │  │
│   │   └── Measures concentration + early buyer advantage                 │  │
│   │                                                                       │  │
│   │   liquidity_risk (25% weight)                                        │  │
│   │   ├── lock_score (40%)                                               │  │
│   │   │   └── How much LP is locked/burned?                             │  │
│   │   ├── removable_flag (30%)                                           │  │
│   │   │   └── Can deployer remove liquidity?                            │  │
│   │   └── low_liq_flag (30%)                                             │  │
│   │       └── Is absolute liquidity dangerously low?                     │  │
│   │                                                                       │  │
│   │   contract_risk (15% weight)                                         │  │
│   │   ├── has_mint_authority (33.33%)                                    │  │
│   │   │   └── Can creator mint more tokens?                             │  │
│   │   ├── has_freeze_authority (33.33%)                                  │  │
│   │   │   └── Can creator freeze wallets?                               │  │
│   │   └── is_honeypot (33.34%)                                           │  │
│   │       └── Is selling restricted?                                     │  │
│   │                                                                       │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Liquidity Risk Calculation

```sql
-- Liquidity risk sub-components
liquidity_risk = (lock_score × 0.4)
               + (removable_flag × 0.3)
               + (low_liq_flag × 0.3)
```

**Lock Score (40% of liquidity risk):**

| LP Secured % | Lock Score | Risk Level |
|--------------|------------|------------|
| 0-30% | 100 | Critical |
| 30-60% | 75 | High |
| 60-90% | 50 | Medium |
| 90-100% | 25 | Low |
| 100% burned | 0 | Minimal |

**Low Liquidity Flag (30% of liquidity risk):**

| Liquidity USD | Flag Value | Risk |
|---------------|------------|------|
| < $10,000 | 100 | Critical |
| $10K - $50K | 50 | Medium |
| > $50,000 | 0 | Low |

### Contract Risk Calculation

```sql
-- Each flag is binary (0 or 1), max contract_risk = 100%
contract_risk = (has_mint_authority × 33.33)
              + (has_freeze_authority × 33.33)
              + (is_honeypot × 33.34)
```

### Risk Thresholds

| Score | Risk Level | UI Color | Recommendation |
|-------|------------|----------|----------------|
| 0-20% | Minimal | Green | Safe to trade |
| 20-40% | Low | Yellow-Green | Caution advised |
| 40-60% | Medium | Yellow | Higher risk |
| 60-80% | High | Orange | Significant risk |
| 80-100% | Critical | Red | Avoid trading |

### Example Calculation

```
Given Token:
- dev_holdings_pct = 15%
- top10_holdings_pct = 25%
- snipers_pct = 5%
- LP 70% locked, 0% burned
- Deployer holds 30% of remaining LP
- Liquidity: $25,000 USD
- has_mint_authority = true
- has_freeze_authority = false
- is_honeypot = false

Step 1: Calculate insider_pct
insider_pct = top10_holdings_pct + snipers_pct = 25 + 5 = 30%

Step 2: Calculate liquidity_risk
lock_score = 50 (70% locked = medium)
removable_flag = 50 (deployer has some LP)
low_liq_flag = 50 ($25K = medium)
liquidity_risk = (50 × 0.4) + (50 × 0.3) + (50 × 0.3) = 50%

Step 3: Calculate contract_risk
contract_risk = (1 × 33.33) + (0 × 33.33) + (0 × 33.34) = 33.33%

Step 4: Calculate rug_risk_pct
rug_risk_pct = (15 × 0.35) + (30 × 0.25) + (50 × 0.25) + (33.33 × 0.15)
             = 5.25 + 7.5 + 12.5 + 5.0
             = 30.25%

Result: LOW risk (20-40% threshold)
```

### ClickHouse Risk Score View

```sql
-- Complete risk score calculation view
CREATE VIEW datastreams.v_token_risk_scores AS
SELECT
    token_mint,
    token_symbol,

    -- ═══════════════════════════════════════════════════════════════════════
    -- RAW RISK COMPONENTS
    -- ═══════════════════════════════════════════════════════════════════════

    dev_holdings_pct,
    top10_holdings_pct,
    snipers_pct,
    (top10_holdings_pct + snipers_pct) AS insider_pct,

    -- ═══════════════════════════════════════════════════════════════════════
    -- LIQUIDITY RISK CALCULATION
    -- ═══════════════════════════════════════════════════════════════════════

    -- Lock score (40% of liquidity risk)
    multiIf(
        lp_burn_pct >= 100, 0,
        (lp_lock_pct + lp_burn_pct) >= 90, 25,
        (lp_lock_pct + lp_burn_pct) >= 60, 50,
        (lp_lock_pct + lp_burn_pct) >= 30, 75,
        100
    ) AS lock_score,

    -- Removable flag (30% of liquidity risk)
    multiIf(
        lp_burn_pct >= 100, 0,
        lp_lock_pct >= 100, 0,
        deployer_lp_pct > 50, 100,
        deployer_lp_pct > 0, 50,
        0
    ) AS removable_flag,

    -- Low liquidity flag (30% of liquidity risk)
    multiIf(
        liquidity_usd < 10000, 100,
        liquidity_usd < 50000, 50,
        0
    ) AS low_liq_flag,

    -- Combined liquidity risk
    (lock_score * 0.4) + (removable_flag * 0.3) + (low_liq_flag * 0.3) AS liquidity_risk,

    -- ═══════════════════════════════════════════════════════════════════════
    -- CONTRACT RISK CALCULATION
    -- ═══════════════════════════════════════════════════════════════════════

    (if(has_mint_authority, 33.33, 0)
     + if(has_freeze_authority, 33.33, 0)
     + if(is_honeypot, 33.34, 0)) AS contract_risk,

    -- ═══════════════════════════════════════════════════════════════════════
    -- FINAL RUG RISK SCORE
    -- ═══════════════════════════════════════════════════════════════════════

    (dev_holdings_pct * 0.35)
    + ((top10_holdings_pct + snipers_pct) * 0.25)
    + (liquidity_risk * 0.25)
    + (contract_risk * 0.15) AS rug_risk_pct,

    -- ═══════════════════════════════════════════════════════════════════════
    -- RISK LEVEL CLASSIFICATION
    -- ═══════════════════════════════════════════════════════════════════════

    multiIf(
        rug_risk_pct >= 80, 'CRITICAL',
        rug_risk_pct >= 60, 'HIGH',
        rug_risk_pct >= 40, 'MEDIUM',
        rug_risk_pct >= 20, 'LOW',
        'MINIMAL'
    ) AS risk_level

FROM datastreams.token_metrics FINAL;
```

### Wallet Risk Classification

Beyond token-level risk, we classify individual wallets:

| Category | Detection Method | Risk Level |
|----------|------------------|------------|
| **Sniper** | Bought within first 10 blocks of token creation | High |
| **Bundler** | Bought 3+ tokens in same block | High |
| **Insider** | Funded from same source as creator | Medium |
| **Phishing** | GoPlus `phishing_activities` flag | Critical |
| **Blacklisted** | GoPlus `sanctioned` or `blacklist_doubt` | Critical |

```sql
-- Wallet risk metrics aggregated per token
SELECT
    token_mint,

    -- Holder risk distribution
    sum(if(is_sniper, balance_tokens, 0)) / total_supply * 100 AS snipers_pct,
    sum(if(is_bundler, balance_tokens, 0)) / total_supply * 100 AS bundler_pct,
    sum(if(is_insider, balance_tokens, 0)) / total_supply * 100 AS insider_pct,
    sum(if(is_phishing OR is_blacklisted, balance_tokens, 0)) / total_supply * 100 AS blacklist_risk_pct

FROM datastreams.token_holders
GROUP BY token_mint, total_supply;
```

---

## 5. Integration: Complete Risk Assessment Query

```sql
-- Complete token risk assessment combining all sources
SELECT
    -- Token identification
    t.token_mint,
    t.token_symbol,
    t.token_name,

    -- Price and volume
    t.price_usd,
    t.market_cap_usd,
    t.stats_24h_volume_usd,

    -- Risk scores
    r.rug_risk_pct,
    r.risk_level,
    r.dev_holdings_pct,
    r.insider_pct,
    r.liquidity_risk,
    r.contract_risk,

    -- Contract flags
    t.has_mint_authority,
    t.has_freeze_authority,
    t.is_honeypot,

    -- External security (from cache)
    s.is_blacklisted AS goplus_blacklisted,

    -- Meta classification
    m.meta_category,
    m.confidence_score AS classification_confidence,

    -- Dev funding source
    f.funded_by AS dev_funded_by,
    f.is_cex_funded AS dev_is_cex_funded,

    -- Composite risk flag
    CASE
        WHEN r.rug_risk_pct >= 80 OR s.is_blacklisted THEN 'AVOID'
        WHEN r.rug_risk_pct >= 60 OR t.is_honeypot THEN 'HIGH_RISK'
        WHEN r.rug_risk_pct >= 40 THEN 'CAUTION'
        ELSE 'ACCEPTABLE'
    END AS trading_recommendation

FROM datastreams.token_metrics FINAL AS t
LEFT JOIN datastreams.v_token_risk_scores AS r ON t.token_mint = r.token_mint
LEFT JOIN datastreams.token_security_cache FINAL AS s ON t.token_mint = s.token_mint
LEFT JOIN datastreams.token_meta_classification FINAL AS m ON t.token_mint = m.token_mint
LEFT JOIN datastreams.wallet_funding_source FINAL AS f ON t.creator = f.wallet

WHERE t.stats_24h_volume_usd >= 10000  -- Active tokens only
ORDER BY t.stats_24h_volume_usd DESC
LIMIT 100;
```

---

## Summary

This part covered the multi-layered risk analysis system:

| Component | Table | Purpose |
|-----------|-------|---------|
| Wallet Funding | `wallet_funding_source` | CEX attribution, sybil detection |
| Token Classification | `token_meta_classification` | AI-powered categorization |
| Security Cache | `token_security_cache` | GoPlus/RugCheck data |
| Risk Scores | `v_token_risk_scores` | Composite rug risk calculation |

**Key Risk Formula:**
```
rug_risk_pct = dev_holdings(35%) + insider(25%) + liquidity_risk(25%) + contract_risk(15%)
```

**External Data Sources:**
- GoPlus: Token security, honeypot detection, wallet blacklists
- RugCheck: Community-driven risk scores, deployer flags
- Solscan Pro: Transaction history for dev funding analysis
- Internal: 507 CEX wallets, 72 MEV tip accounts, 51 platform fee wallets

---

## Next in the Series

**Part 9: Platform Features - Orders, Rewards & Referrals** - We'll explore Click platform-specific features including order tracking, user rewards, and referral earnings systems.
