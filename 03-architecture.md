# Click Technical Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CLICK TECHNICAL ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  EXTERNAL DATA SOURCES                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                            │
│  │  Solana Geyser  │    │   Hyperliquid   │                            │
│  │  (Blockchain)   │    │      API        │                            │
│  └────────┬────────┘    └────────┬────────┘                            │
│           │                      │                                      │
│           ▼                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    BARE METAL INFRASTRUCTURE                     │   │
│  │                                                                  │   │
│  │  ┌──────────────┐    ┌──────────────────┐                       │   │
│  │  │Solana Engine │    │Hyperliquid Indexer│                      │   │
│  │  │  (Indexer +  │    │    (Indexer)      │                      │   │
│  │  │  Execution)  │    │                   │                      │   │
│  │  └──────┬───────┘    └────────┬──────────┘                      │   │
│  │         │                     │                                  │   │
│  │         ▼                     ▼                                  │   │
│  │  ┌────────────────────────────────────────────────────────┐     │   │
│  │  │                      NATS                               │     │   │
│  │  │              (Central Message Bus)                      │     │   │
│  │  └───────────────────────┬────────────────────────────────┘     │   │
│  │                          │                                       │   │
│  │         ┌────────────────┼────────────────┐                     │   │
│  │         ▼                ▼                ▼                     │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                │   │
│  │  │DataStreams │  │ ClickHouse │  │   Redis    │                │   │
│  │  │(Real-time  │  │(Historical │  │  (Cache)   │                │   │
│  │  │ metrics)   │  │  storage)  │  │            │                │   │
│  │  └────────────┘  └────────────┘  └────────────┘                │   │
│  │                                                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                          │                                              │
│                          ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       AWS CLOUD                                   │  │
│  │                                                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │  │
│  │  │   Signer     │  │   REST API   │  │   Postgres   │           │  │
│  │  │(Nitro Enclave│  │ (CloudFront) │  │   (Aurora)   │           │  │
│  │  │   TEE)       │  │              │  │              │           │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                          │                                              │
│                          ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      PRODUCTS (Clients)                          │  │
│  │                                                                   │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │  │
│  │  │Telegram  │ │Telegram  │ │   Web    │ │ Chrome   │ │ Wallet │ │  │
│  │  │  Bot     │ │  Perps   │ │ Terminal │ │Extension │ │        │ │  │
│  │  │  (AWS)   │ │  (AWS)   │ │ (Vercel) │ │ (Local)  │ │(Mobile)│ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────┘ │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure & Hosting

| Component | Environment | Type | HA Status |
|-----------|-------------|------|-----------|
| **Solana Engine** | Bare Metal | Service | Single instance (SPOF) |
| **Hyperliquid Indexer** | Bare Metal | Service | Single instance (SPOF) |
| **DataStreams** | Bare Metal | Service | HA |
| **ClickHouse** | Bare Metal | Database | Cluster |
| **NATS** | Bare Metal | Message Bus | Cluster |
| **Redis** | Bare Metal | Cache | Single instance (SPOF) |
| **Signer** | AWS (Nitro Enclave) | Service | HA |
| **REST API** | AWS (CloudFront) | Service | HA |
| **Postgres** | AWS (Aurora) | Database | HA |
| **Telegram Bot** | AWS | Product | HA |
| **Telegram Perps** | AWS | Product | HA |
| **Web Terminal** | Vercel | Product | HA |
| **Chrome Extension** | Local | Product | N/A |
| **Wallet** | Local/Mobile | Product | N/A |

---

## Component Details

### 1. Solana Engine (Bare Metal)

**Role A - Indexer**:
- Ingests Solana data via Geyser
- Indexes into structured events (Swaps, Transfers)
- Broadcasts to NATS

**Role B - Execution**:
- Listens for order commands from NATS
- Executes trades on Solana
- Returns status to NATS

**Dependencies**: None (Source of Truth)

**NATS Topics Published**:
- `solana.trades.v3`
- `solana.transfers.v2`
- `solana.tokens.v2`
- `solana.pools.v1_0_0`
- `solana.liquidity_events.v1`
- `solana.token_events.v1`
- `solana.graduation_events.v1`
- `solana.failed_transactions.v1`

### 2. Hyperliquid Indexer (Bare Metal)

**Role**: Index Hyperliquid perps data

- Ingests from Hyperliquid API
- Indexes events (Liquidations, Fills, Funding)
- Broadcasts to NATS

**Note**: Indexer only - order submission via REST API → Hyperliquid API directly

**Dependencies**: None (Source of Truth)

**NATS Topics Published**:
- `hyperliquid.fills`
- `hyperliquid.deposits`
- `hyperliquid.withdrawals`
- `hyperliquid.funding`
- `hyperliquid.liquidations`

### 3. DataStreams (Bare Metal, HA)

**Role**: Stateful market data processor

- Consumes events from NATS
- Computes derived state:
  - OHLCV candles (14 timeframes)
  - Price, market cap, volume
  - Price changes (5m, 1h, 6h, 24h)
  - Holder analytics (count, top 10%, dev%)
  - Wallet classifications (sniper, insider, bundler)
  - Rug risk scores
  - Whale detection
- Publishes derived data to NATS

**Performance**: p95 < 5ms from event to publication

**Dependencies**: Solana Engine, Hyperliquid Indexer (via NATS)

**NATS Topics Published**:
- `solana.enriched_trades`
- `solana.market_ohlcv.*`
- `solana.token_ohlcv.*`
- `solana.token_metrics.v2`
- `solana.discovery.*`
- `click.order_overview`
- `solana.position_overview`

### 4. ClickHouse (Bare Metal, Cluster)

**Role**: Historical data storage and analytics

- Ingests all events from NATS
- Permanent storage layer
- Serves analytical queries

**Performance**:
- Ingestion: 100,000+ events/sec
- Ingestion lag: p95 < 1 second
- Query latency: p95 < 250ms

**Storage**:
- Hot tier: NVMe (recent data)
- Cold tier: S3 (historical)

**Dependencies**: None (infrastructure layer)

**Consumed Topics**: All from Solana Engine, Hyperliquid Indexer, DataStreams

### 5. NATS (Bare Metal, Cluster)

**Role**: Central message bus for async communication

**Publishers**:
- Solana Engine (blockchain events, order status)
- Hyperliquid Indexer (perps events)
- DataStreams (derived state)

**Subscribers**:
- DataStreams (raw events)
- ClickHouse (all events for storage)
- REST API (market data, order status)

**Dependencies**: None (infrastructure layer)

### 6. Redis (Bare Metal)

**Role**: In-memory cache

**Use Cases**:
- REST API: Rate limiting, verification codes
- Telegram Bot: Session state, conversation context
- Telegram Perps: Session state

**Dependencies**: None (infrastructure layer)

### 7. Signer (AWS Nitro Enclave, HA)

**Role**: Non-custodial wallet and signing service

- TEE-based key management
- Transaction signing
- Authentication and intent tokens

**Callers**:
- Solana Engine: Sign transactions for order execution
- REST API: Validate wallets, signup, manage intent tokens

**Security**: Keys never stored in raw form

**Dependencies**: None (foundational security)

### 8. Postgres (AWS Aurora)

**Role**: User data storage

**Stores**:
- User profiles
- Orders
- Account state
- Settings
- Wallet associations

**Access**: REST API queries

### 9. REST API (AWS CloudFront)

**Role**: Primary gateway for all products

**Functions**:
- Solana Order Flow: HTTP → NATS → Solana Engine
- Hyperliquid Order Flow: HTTP → Hyperliquid API directly
- Real-Time Flow: NATS (DataStreams) → WebSocket to clients
- Data Retrieval: Postgres (user data), ClickHouse (analytics)

**Communication**:
- Telegram Bots: HTTP + NATS + Redis
- Web/Extension/Wallet: HTTP + WebSocket

---

## Data Flow

### 1. Ingestion (Blockchain → Click)

```
Solana Geyser → Solana Engine → NATS
Hyperliquid API → HL Indexer → NATS
```

### 2. Processing (Raw → Derived)

```
NATS → DataStreams → Computes:
    • OHLCV candles (14 timeframes)
    • Price, market cap, volume
    • Price changes (5m, 1h, 6h, 24h)
    • Holder count, top 10 %, dev %
    • Sniper %, insider %, bundler %
    • Rug risk score
    • Whale detection
    • Enriched trades with wallet badges
DataStreams → NATS (publishes derived data)
```

### 3. Storage (Permanent Record)

```
NATS → ClickHouse (historical queries)
REST API → Postgres (user data)
```

### 4. Serving (To Products)

```
REST API ← ClickHouse (historical data)
REST API ← NATS (real-time data)
REST API ← Postgres (user data)
REST API → WebSocket → Products
```

### 5. Execution (Trades)

```
Product → REST API → NATS → Solana Engine → Blockchain
Product → REST API → Hyperliquid API (direct for perps)
```

---

## Infrastructure & Operations

| Concern | Solution |
|---------|----------|
| **Observability** | Prometheus (metrics), Loki (logs), Tempo (tracing), Grafana (dashboards) |
| **Networking** | AWS Direct Connect between Bare Metal ↔ AWS |
| **CDN/Protection** | CloudFront in front of REST API |
| **Secrets** | AWS Secrets Manager |
| **Backups** | Aurora automated, ClickHouse → S3 |
| **Resilience** | Retry with exponential backoff for external APIs |

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Data refresh latency | <100ms |
| Chart load time | <0.5 seconds |
| Page load time | <1 second |
| Order execution | <2 seconds |
| Transaction success rate | >99% |
| System uptime | 99.9% |
| DataStreams processing | p95 < 5ms |
| ClickHouse ingestion lag | p95 < 1 second |
| ClickHouse query latency | p95 < 250ms |

---

## Known Limitations / SPOFs

| Component | Status | Risk | Mitigation |
|-----------|--------|------|------------|
| Solana Engine | Single instance | Order execution SPOF | Planned HA |
| Hyperliquid Indexer | Single instance | HL data ingestion SPOF | Planned HA |
| Redis | Single instance | Session/cache SPOF | Planned cluster |

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| **Blockchain Indexing** | Solana Geyser, Custom Rust indexer |
| **Message Bus** | NATS (JetStream) |
| **Real-time Processing** | DataStreams (Custom Rust) |
| **Analytics Database** | ClickHouse |
| **User Database** | PostgreSQL (Aurora) |
| **Cache** | Redis |
| **Signing** | AWS Nitro Enclave (TEE) |
| **API Gateway** | CloudFront + Custom REST API |
| **Web Frontend** | React / Next.js (Vercel) |
| **Telegram Bots** | Node.js |
| **Chrome Extension** | TypeScript |
| **Observability** | Prometheus, Loki, Tempo, Grafana |
