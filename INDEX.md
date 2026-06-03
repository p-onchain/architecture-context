# BlackSwan Exchange — Architecture Context

> Lightweight service map for AI coding agents. Tells you WHERE to find things, not HOW they work internally — clone the repo and read the code for that.

## System Overview

BlackSwan is a crypto exchange platform (Paribu v6). Go microservices, CQRS/Event Sourcing, gRPC (sync) + Redpanda/Kafka (async). Deployed on Kubernetes (EKS) via ArgoCD GitOps.

## Service Map

| Domain | Services | Repos |
|--------|----------|-------|
| **Trading** | Order entry, matching engine, response aggregation, conditional triggers, event persistence | order-api, match-core, order-responder, conditional-order, match-forge |
| **Wallet** | Balance management, ledger persistence, validation | wallet, wallet-ledger-sink, wallet-validator, wallet-outbox |
| **Transaction** | Crypto/fiat deposit & withdraw, blockchain monitoring | transaction, block-listener, onchain (TS) |
| **User & Auth** | User management, KYC, auth (OIDC/OAuth2/passkeys/SSO) | user-service, mellon |
| **Market Data** | External prices, market metadata, per-market config | global-price-tracker, market-api, market-configs |
| **Notification** | Push/SMS/email/in-app delivery | notif2, notification |
| **Admin** | Back-office UI, dynamic config, feature toggles | gopanel, config-service, feature-flags |
| **Compliance** | AML screening, custody, reconciliation | sanctions, elliptic-screener, custody-integration |
| **Finance** | Invoicing, bank integration, commission processing | invoice-service, bank-integration, commission-update-worker |
| **Staking** | Pool-based crypto staking | staking-service |
| **Campaign** | Promotions, coupon campaigns | campaign-service |
| **Read Models** | ~25 CQRS projection/query pairs for all domains | read-mono (monorepo) |
| **Gateway** | API gateway, BFF for web, BFF for mobile | krakend-gateway, bff-api, bff-client |
| **Real-time** | WebSocket streaming | ws-hub (web/mobile), wapi (API-key traders) |
| **Events** | Transaction event archival/mirroring | eventificator |
| **Infra** | GitOps, Terraform, Helm | blackswan-gitops, blackswan-infrastructure, platform-gitops, platform-terraform |

All core service repos are under **p-blackswan** org unless noted otherwise.

## Request Flow

```
Client (Samaritan / Web / WAPI)
  → KrakenD Gateway (JWT validation, rate limiting, API-key auth)
    → BFF (bff-api for web, bff-client for mobile)
      → gRPC microservices
        → Redpanda/Kafka (async events)
          → read-mono projections → query services → back to BFF
```

## Trading Flow (Core Path)

```
bff → order-api → wallet (reserve funds) → match-core-{market} (match)
  ↓ Kafka: order.events.match.{market}, order.events.status.{market}
  → wallet (settle trades) → ledger-logs → wallet-ledger-sink
  → order-responder (aggregate response)
  → match-forge (persist match events)
  → read-mono projections (orderbook, ticker, balance, etc.)
```

## Key Kafka Topics

| Topic Pattern | Producer → Consumers |
|---------------|---------------------|
| `order.events.match.{market}` | match-core → wallet, order-responder, match-forge, read-mono |
| `order.events.status.{market}` | match-core → order-responder, conditional-order, read-mono |
| `ledger-logs` | wallet → wallet-ledger-sink, read-mono |
| `orderbook.match_price` | match-core → order-api, conditional-order, read-mono |
| `orderbook.state` | match-core → read-mono (orderbook) |
| `config-service-events` | config-service → order-api, read-mono |
| `user-commission-events` | commission → commission-update-worker, wallet |

## Client Applications

| Client | Repo | Stack | BFF |
|--------|------|-------|-----|
| **Samaritan** (mobile) | pikachu-exchange/samaritan | React Native (Expo) | bff-client |
| **Web** | p-blackswan/web | Vue.js | bff-api |
| **WAPI** (trading API) | p-blackswan/wapi | Go WebSocket | Direct (API-key via KrakenD) |
| **GoPanel** (admin) | pikachu-exchange/gopanel | Go + Vue | Direct |

## Shared Libraries

| Library | Repo | What it gives you |
|---------|------|-------------------|
| **corekit** | p-blackswan/corekit | DB, Kafka, gRPC, config, error handling, worker patterns |
| **proto-hub** | p-blackswan/proto-hub | All protobuf definitions + generated Go code |
| **observer** | p-blackswan/observer | OTEL exporters, Pyroscope, metrics/tracing |
| **fixedpoint** | p-blackswan/fixedpoint | Decimal arithmetic for financial calculations |

## Infrastructure

- **Cloud:** AWS (multi-account, Terragrunt)
- **K8s:** EKS + ArgoCD
- **Messaging:** Redpanda (Kafka-compatible)
- **DB:** PostgreSQL (primary), ClickHouse (analytics), Redis (cache)
- **Gateway:** KrakenD + custom Go plugins
- **Observability:** SigNoz (traces/logs), VictoriaMetrics (metrics), Pyroscope (profiling)
- **CI/CD:** GitHub Actions → Docker → ArgoCD
- **Feature Flags:** flagd (OpenFeature)
- **Secrets:** HashiCorp Vault

## GitHub Orgs

| Org | What's there |
|-----|-------------|
| **p-blackswan** | Core exchange services, infra, shared libs |
| **pikachu-exchange** | Mobile app (Samaritan), gopanel, legacy services |
| **p-utilities** | Internal tools, KYC SDK, charting, storybook |
| **p-onchain** | On-chain/DeFi related services |

## How to Use This

1. Read this INDEX for the big picture
2. Read `services/<name>.md` for the service you're touching — it tells you what it talks to
3. Read `flows/<flow>.md` if you need an end-to-end process
4. Read `conventions.md` for coding standards
5. **Clone the repo and read the code** for implementation details
