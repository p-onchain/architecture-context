# Supporting Services

> Quick reference for supporting/smaller services. For details, clone the repo.

---

## Trading Support

**order-responder** — Aggregates match/status events from Kafka, builds order response for fast-cancel flow. Shares Redis order cache (DB 7) with order-api.

**match-forge** — Persists match events from Redpanda to PostgreSQL as append-only event store. S3 archival for old data.

**conditional-order** (gRPC :50051) — Stop-loss, take-profit, conditional triggers. Consumes `orderbook.match_price`, calls order-api to submit triggered orders.

## Config & Feature Management

**config-service** — Dynamic config management (namespace **`shelby`**, HTTP **:8080**, REST not gRPC). Produces `config-service-events` to Kafka (outbox worker). Market list via `GET /default?basic=true` (markets live only in the `basic` blob); each market carries `precisions`+`steps` (no price, no limits). Tradeable = `unlisted!=true && suspended` absent. Per-market user override via `GET /merged/{userID}`.

**feature-flags** — YAML-based feature flag definitions consumed by flagd/OpenFeature.

**market-configs** — YAML per market definition. ArgoCD watches and generates Deployments.

## Notification

**notif2** / **notification** — Multi-channel delivery (push, SMS, email, in-app). notif2 is the newer version.

**alarm-service** — User-defined price alarms and custom alert rules.

## Finance & Compliance

**commission-update-worker** — Consumes `user-commission-events`, writes rates to Redis for wallet.

**bank-integration** — Bank API integration for fiat operations.

**custody-integration** — HSM/custody signing bridge for crypto transactions.

**elliptic-screener** — AML/compliance screening.

**sanctions** — Sanctions list checking.

**invoice-service** — Invoice generation for trades/transactions.

## Blockchain

**block-listener** — Blockchain event listener for deposit detection and tx confirmation.

**onchain** (TypeScript) — Blockchain monitoring, deposit detection.

## Staking & Campaign

**staking-service** — Pool-based crypto staking (create, stake, redeem, distribute rewards).

**campaign-service** — Promotions, coupon campaigns, referral rewards.

## Events & Data

**eventificator** — Transaction event archival. Produces `archive.transaction-events` topic (synthesized history + live mirror).

**global-price-tracker** — Fetches/serves global crypto prices from external sources.

**heimdall** (Rust, gRPC :50099) — External data hub (CoinGecko, sentiment indices, etc.).

## Real-time

**ws-hub** — WebSocket hub for web/mobile clients. JWT auth.

**wapi** — Low-latency WebSocket for API-key traders. Direct Redpanda consumption. See `clients/wapi.md`.
- As of 2026-06: uses **AsyncCommit** offset policy (`PROCESSOR_COMMIT_POLICY=async`) — removes per-message broker round-trip while preserving at-least-once.
- As of 2026-06: **per-partition head-of-line isolation** enabled (`PROCESSOR_PARTITION_ISOLATION=true`) — slow partitions are paused at broker instead of wedging the poll loop.

## Admin

**gopanel** (pikachu-exchange) — Back-office admin panel. Go backend + Vue frontend.

## Integration

**recurring-api** — Recurring buy (DCA) service, CQRS + Event Sourcing.

**partner-integration** — External partner integrations.

**kep-service** — KEP (legal registered email) integration.

**mkk** — MKK regulatory reporting integration.

**input-validator** (gRPC :50081) — Input validation service (addresses, amounts, etc.).
