# BlackSwan Exchange — Architecture Context

> Lightweight overview for AI coding agents. Read this first.
> For details on a specific service, see `services/<name>.md`.
> For end-to-end flows, see `flows/`.

## System Overview

BlackSwan is a crypto exchange platform (Paribu v6) built as Go microservices with CQRS/Event Sourcing. Services communicate via gRPC (sync) and Redpanda/Kafka (async). All services are deployed on Kubernetes (EKS) via ArgoCD GitOps.

## Domain Map

| Domain | Services | Purpose |
|--------|----------|---------|
| **Trading** | order-api, match, order-responder, conditional-order, match-forge | Order entry, matching, response aggregation, conditional triggers, event persistence |
| **Wallet** | wallet, wallet-ledger-sink, wallet-validator, wallet-asset-cleanup | Balance management, ledger persistence, validation, cleanup |
| **Transaction** | transaction, onchain | Crypto/fiat deposit & withdraw, blockchain monitoring |
| **User** | user-service, mellon (auth) | User management, KYC, auth (OIDC/passkeys) |
| **Market Data** | global-price-tracker, market-api, market-configs | External prices, market metadata, per-market config (GitOps) |
| **Notification** | notif2, notification | Push/SMS/email/in-app delivery |
| **Admin** | gopanel, config-service, feature-flags | Back-office UI, dynamic config, feature toggles |
| **Compliance** | sanctions, elliptic-screener, custody-integration, reconciliation | AML screening, custody, reconciliation |
| **Finance** | invoice-service, bank-integration, commission-update-worker, commission-cost-tracker | Invoicing, bank integration, commission processing |
| **Staking** | staking-service | Pool-based crypto staking |
| **Campaign** | campaign-service | Promotions, coupon campaigns |
| **Read Models (CQRS)** | read-mono (monorepo) | ~25 projection/query pairs for all domains |
| **Gateway** | krakend-gateway, bff-api, bff-client | API gateway, BFF for web, BFF for mobile |
| **Real-time** | ws-hub, wapi | WebSocket streaming for web/mobile, API-key WS for traders |
| **Observability** | observer (lib), alarm-service | Shared OTEL lib, price/custom alarms |
| **Infrastructure** | blackswan-gitops, blackswan-infrastructure, blackswan-helm-base | GitOps (ArgoCD), Terraform/Terragrunt, Helm base chart |

## Request Flow (Simplified)

```
Client (Samaritan/Web)
  → KrakenD Gateway (JWT validation, rate limiting, routing)
    → BFF (bff-api for web, bff-client for mobile)
      → gRPC microservices (order-api, wallet, user-service, etc.)
        → Redpanda/Kafka (async events)
          → read-mono projections (update query stores)
            → query services (serve read requests back to BFF)
```

## Trading Flow (Core Path)

```
bff → order-api (HTTP :9000 + gRPC :60060)
        → wallet (gRPC :50058) — reserve funds
        → match-{market} (gRPC :50059) — submit to matching engine
            ↓ (Kafka: order.events.match.{market}, order.events.status.{market})
        → order-responder — aggregates match/status events, builds response
        → wallet — settles trades (consumes match events)
            ↓ (Kafka: ledger-logs)
        → wallet-ledger-sink — persists ledger to PostgreSQL
        → match-forge — persists match events to PostgreSQL
```

## Key Kafka Topics

| Topic Pattern | Producer | Consumers | Schema |
|---------------|----------|-----------|--------|
| `order.events.match.{market}` | match engine | wallet, order-responder, match-forge, read-mono | Protobuf (Schema Registry) |
| `order.events.status.{market}` | match engine | order-responder, conditional-order, read-mono | Protobuf |
| `ledger-logs` | wallet | wallet-ledger-sink, read-mono (balance, financial-history) | Protobuf |
| `user-commission-events` | commission system | commission-update-worker, wallet | Protobuf |
| `orderbook.match_price` | match engine | order-api, conditional-order, read-mono (ticker) | Protobuf |
| `orderbook.state` | match engine | read-mono (orderbook) | Protobuf |
| `config-service-events` | config-service | order-api, read-mono | Protobuf |
| `external.ledger-logs` | redpanda-connect (mirror) | pikachu-exchange consumers | Protobuf |
| `external.commission` | redpanda-connect (mirror) | pikachu-exchange consumers | Protobuf |

## Key gRPC Service Ports (Local Dev)

| Service | gRPC Port | HTTP Port | Notes |
|---------|-----------|-----------|-------|
| order-api | :60060 | :9000 | Both HTTP and gRPC |
| wallet | :50058 | — | |
| match-{market} | :50059 | — | One instance per market |
| conditional-order | :50052 | — | |
| bff-api | — | :3001 | HTTP only, calls gRPC downstream |
| bff-client | — | :8080 | HTTP only, calls gRPC downstream |
| user-service | :50074 | — | Command + Query |
| ticker-query | :50073 | — | |
| balance-query | :50070 | — | |
| financial-history-query | :50071 | — | |
| klines-query | :50072 | — | |
| orderbook-query | :50075 | — | |
| transaction | :50076 | — | |
| commission-api | :50077 | — | |
| uservolume-query | :50078 | — | |
| global-price-tracker | :50079 | — | |
| notification-api | :50080 | — | |
| input-validator | :50081 | — | |
| transaction-query | :50082 | — | |
| user-state-query | :50084 | — | |
| favourite-query | :50086 | — | |
| open-order-query | :50087 | — | |
| heimdall | :50099 | — | External data hub (Rust) |

## Shared Libraries

| Library | Repo | Purpose |
|---------|------|---------|
| **corekit** | p-blackswan/corekit | Standard Go toolkit — DB (Postgres/Redis/ClickHouse), Kafka consumer/producer, gRPC client/interceptors, env config, error handling, retry, logging, worker patterns |
| **observer** | p-blackswan/observer | Shared observability — OTLP exporters, Pyroscope profiling, metrics/tracing setup |
| **proto-hub** | p-blackswan/proto-hub | Central protobuf definitions + generated Go code for all service contracts |
| **feature-flags-go** | p-blackswan/feature-flags-go | Feature flag client library |
| **fixedpoint** | p-blackswan/fixedpoint | Decimal arithmetic for financial calculations |

## Infrastructure

- **Cloud:** AWS (multi-account via Terragrunt)
- **Kubernetes:** EKS clusters managed via ArgoCD
- **Messaging:** Redpanda (internal, Kafka-compatible) + MSK (external bridge)
- **Databases:** PostgreSQL (primary), ClickHouse (analytics), Redis (cache/state)
- **Schema Registry:** Redpanda Schema Registry (Protobuf schemas)
- **Gateway:** KrakenD (with custom Go plugins for API-key auth + request logging)
- **Observability:** SigNoz (traces/logs), VictoriaMetrics (metrics), Pyroscope (profiling), Grafana (dashboards)
- **CI/CD:** GitHub Actions (shared-workflows) → Docker → ArgoCD GitOps
- **Feature Flags:** flagd (OpenFeature) via feature-flags repo (YAML)
- **Secrets:** HashiCorp Vault

## CQRS Pattern (read-mono)

BlackSwan uses CQRS extensively. Write services (wallet, user-service, transaction, etc.) emit events to Kafka. The `read-mono` monorepo contains ~25 projection/query service pairs:

Each domain in read-mono has:
- **projection** — Kafka consumer that builds a read-optimized view in PostgreSQL/ClickHouse
- **query** — gRPC server that serves the read-optimized view

Domains in read-mono: alarm, anomaly-detection, balance, bank-integration, campaign, commission, cost-basis, custody-integration, favourite, feedback, financial-history, klines, open-order, orderbook, orderbook-wapi, pass, pnl, pnl-agg, staking, ticker, transaction, user, user-state, uservolume, ws

## Match Engine Topology

Each market (e.g., `btc-try`, `eth-try`, `sol-try`) runs as a separate match engine instance. Market definitions are in `p-blackswan/market-configs` (GitOps — YAML per market). ArgoCD generates one Deployment per market YAML file.

Match engines are stateful — they maintain an in-memory order book and produce events to per-market Kafka topics (`order.events.match.{market}`, `order.events.status.{market}`).

## Client Applications

| Client | Repo | Stack | BFF |
|--------|------|-------|-----|
| **Samaritan** (mobile) | pikachu-exchange/samaritan | React Native (Expo) | bff-client |
| **Web** | p-blackswan/web | Vue.js | bff-api |
| **WAPI** (trading API) | p-blackswan/wapi | Go WebSocket server | Direct (API-key auth via KrakenD) |
| **Desktop** | p-utilities/prb-desktop | Vue (Electron/Tauri) | bff-api |
| **GoPanel** (admin) | p-blackswan/gopanel | Go backend + Vue frontend | Direct (internal) |

For client-specific details, see `clients/`.

## GitHub Organizations

| Org | Purpose |
|-----|---------|
| **p-blackswan** | Core exchange services, infrastructure, shared libs |
| **pikachu-exchange** | Mobile app (Samaritan), legacy services, financial-history |
| **p-utilities** | Internal tools, KYC, charting, storybook, SDK packages |

## How to Use This Context Pack

If you're an AI agent working on a specific service:
1. Read this INDEX.md for the big picture
2. Read `services/<service-name>.md` for the service you're working on
3. Read `flows/<relevant-flow>.md` if you need to understand an end-to-end process
4. Read `conventions.md` for coding standards and patterns
5. Check `proto-hub` for the exact gRPC contract definitions
