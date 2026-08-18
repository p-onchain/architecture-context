# Supporting Services

> Quick reference for supporting/smaller services. For details, clone the repo — and read its
> `.cd/helm/<env>/*.yaml` for ports, namespace and dependencies.
> **Last verified 2026-08-17** against live prod pods + the p-blackswan repo inventory.
> `ns` = the namespace pods actually run in; `HW` marks the Huawei cluster `exc-prod-hw`.

---

## Trading Support

**order-responder** (`ns blackswan`, ~64 pods) — aggregates match/status events from the internal
Redpanda into the extended order response; writes it to its own ElastiCache
(`exc-prod-order-responder`) which order-api polls. Also owns the `client_order:` keyspace.
Has its own repo (`p-blackswan/order-responder`) — the old "planned, not deployed" note is wrong.

**order-strategies** (`ns blackswan`, 5 pods) — order strategy service; reached by both BFFs as
`SERVICE_STRATEGY_API_URL`.

**match-forge** — persists match/status/orderbook/WAL events to PostgreSQL as an append-only event
store; S3 archival for old data. Consumes `order.requests.*` too.

**conditional-order** (gRPC :50051, `ns blackswan`) — stop-loss / take-profit / conditional triggers.
Consumes `orderbook.match_price`, produces `order.events.status`, calls order-api's `Execute` callback.

**market-api** (`ns blackswan`) — market lifecycle API; provisions the per-market topics via market CR sync.

**market-configs** — YAML source of truth per market; its own ApplicationSet generates one match
engine per file (**263 in prod**).

**market-simulator** — synthetic market activity for the internal test exchange.

## Config & Feature Management

**config-service** (`ns shelby`, HTTP **:8080**, REST not gRPC) — dynamic config. Produces
`config-service-events` (outbox worker `config-service-worker`). Market list via
`GET /default?basic=true` (markets live only in the `basic` blob); each market carries
`precisions`+`steps` (no price, no limits). Tradeable = `unlisted != true && suspended` absent.
Per-user override via `GET /merged/{userID}`.

**flipt-v2** (`ns blackswan`, HTTP :8080 / gRPC :9000, `flipt.int.paribu.com`) — **the** feature-flag
service. Flag state lives in `p-blackswan/feature-flags` under `flipt/<env>`; Go client
`feature-flags-go`; dashboard `ff-dash`. ⚠️ flagd/OpenFeature is gone.

## Notification

See **`services/notification.md`** — notif2 (`notify-api`) + read-mono `notify` (HW) + legacy
`notification` v1.

**alarm-service** (`ns corleone`) — user price alarms: `alarms-server` (**:50082**),
`quick-alarms-worker`, `custom-alarms-worker`; read side `alarm-query` (read-mono).

## Finance & Compliance

**commission-update-worker** (`ns blackswan`) — consumes `user-commission-events`, writes rates to
Redis for wallet. **commission-transfer-job** is a separate repo/app.

**bank-integration** (HW) + **bank-integration-stock** — fiat bank APIs; read side
`bank-integration-query` (read-mono, HW).

**bff-instant-pay** (both clusters) and **SoapInstantPay** (C#, HW) — instant fiat pay-in.

**custody-integration** (`ns shelby`) — HSM/custody signing bridge; read side
`custody-integration-query`. **legacy-custody-bff** (from the transaction repo) still runs on AWS.

**elliptic-screener** (`ns shelby`) — AML screening (`elliptic-screener-scheduler` + `-worker`),
produces `elliptic-screener-events`. **sanctions** (HW) — sanctions list checking.

**invoice-service** (HW) — invoicing (`invoice-service-api` + `-worker`).
**reconciliation** (HW) — `reconciliation-server` + `reconciliation-job`.
**pdf-operations** (Python/FastAPI, HW) — S3 PDF processing.

**mkk** — MKK regulatory reporting. ⚠️ Repo **archived** and no pods are running, though a stale
ArgoCD app still exists.

**kep-service** (`ns shelby`, app `kep`) — KEP (legal registered e-mail) integration for Legal.

**paribu-one-compliance** (apps `service` + `frontend`) — compliance panel, forked from p-backoffice.

**audit-log-api** — audit log query & ingest; KrakenD request logs into TimescaleDB.

## Rewards, Staking & Campaign

**reward-service** (HW, `ns shelby`) — `reward-service`, `reward-consumer`, `reward-distributor`,
`reward-jobs`; read side `reward-query` (read-mono, its own `exc-prod-reward` DB).

**staking-service** (`ns corleone`) — `staking-service`, `staking-distributor`, `staking-jobs`;
read side `staking-query`. Produces `staking-events`. Fire-and-forget notify caller.

**campaign-service** (`ns shelby`) — promotions/coupons: `campaign-service` +
`campaign-outbox-worker`; read side `campaign-projection`/`-query` (read-mono).

## Blockchain / DeFi

**onchain** + **bundler-erc4337** — see **`services/onchain.md`**.

**block-listener** — blockchain deposit detection / confirmations (legacy path alongside onchain).

## Users, Auth & KYC

**estel** (Rust) — the KYC platform: `kyc-api`, `kyc-orchestrator`, `kyc-backoffice`
(`kyc.internal.paribu.com`, provisioned partly via `kyc-ansible` on EC2). Client side:
`p-utilities/p-kyc`, `nitro-kyc`, `react-native-nfc-passport-reader`.

**tier-worker** (HW, from user-service) — user tier computation.

**gatekeeper-service** — access gatekeeping (no repo description; ArgoCD app exists, no steady-state
pods in exc-prod-* — clone it before assuming it's live).

## Events & Data

**eventificator** — transaction event archival/mirroring (`eventificator-mirror`,
`eventificator-backfill`, plus `mirror`/`backfill` apps). Produces `archive.transaction-events`.

**redpanda-connect** (`ns blackswan`, ~64 pods) — the Redpanda↔MSK bridge. Config lives in
platform-gitops, **not** in a service repo. See `INDEX.md`.

**log-timestamp** — `log-timestamp-daily` / `-single` / `-backfill` (job-shaped, so no steady pods).

**global-price-tracker** (`ns saul`) — external crypto prices (`-api` + `-worker`).

**heimdall** (Rust, `ns momentum`, gRPC :50051 via `heimdall-headless`) — external data hub
(CoinGecko, sentiment, indices). Uses ClickHouse Cloud + RDS `exc-prod-momentum` + **Valkey**, and
pulls onchain's token catalog from `defi-api:8080/v1/tokens/catalog`.

**search** / **in-app-search-engine** — search services (thin descriptions; clone).

## Real-time

**ws-hub** (`ns corleone`) — WebSocket hub for web/mobile, JWT auth. Consumes `websocket-events`
(group `ws-gateway-group`); the fan-in is produced by balance-projection, ws-projection,
config-service, transaction, financial-history-projection, orderbook-projection and ticker-worker.

**wapi** (`ns corleone`) — low-latency WebSocket for API-key traders, straight off the internal
Redpanda (`order.events.match`, `order.events.status`, `orderbook.match_price`, `ledger-logs`,
`wallet-error-events`). See `clients/wapi.md`.

## Admin, Support & Analytics

**gopanel** (`ns gopanel`) — back-office panel (`gopanel-backend` + `gopanel-frontend`,
`go-v6.ceteris.io`). ⚠️ Now **p-blackswan/gopanel** — the equities/exchange carve-out of Paribu One.
The DeFi admin panel is **p-backoffice/paribu-one-defi**, not here.

**crm-service** (`ns gopanel`) — `crm-api`, `crm-backoffice-api`, `crm-processor`, `crm-scheduler`,
`crm-outbox-worker`. Drives bulk notification campaigns.

**dash** (`ns gopanel`) — analytics: `dash-api`, `dash-frontend`, `dash-projection` (15 pods),
`dash-recomputer`.

**helpdesk-paribu** + **assistant-paribu** (`ns gopanel`) — `helpdesk-backend` (:8080, still called
`TICKBU_API_URL` by the BFFs), `helpdesk-frontend`, `helpdesk-assistant` (:8090, AI). Core lives in
`p-backoffice/helpdesk-core` / `helpdesk-frontend`.

**support-web**, **webhelp** (`ns gopanel`) — public support/help surfaces.

**anomaly-detection** (read-mono, `ns gopanel`) — projection + query + worker over
`orderbook.analysis.snapshot` and trade data.

**clamav** (`ns gopanel`) — attachment virus scanning (`MaxFileSize` 30 MiB gates the helpdesk upload cap).

**Paribu One suite** (p-backoffice) — `paribu-one` framework (host/core/pkg/system apps) plus domain
repos `paribu-one-defi`, `-denetim`, `-dys`, `-nexus`, `-chat`, `-saasops`, `-compliance`, and the
internal **kanban** app. Domain repos use a `framework/` git submodule.

## Integration & Misc

**partner-integration** (HW, `ns corleone`) — external partners (Beşiktaş & Mohikan);
produces `partner-integration-events`.

**temp-enqura-proxy** (`ns blackswan`) — temporary Enqura proxy for the Stock product.

**kvmks** (HW, `ns blackswan`) — KVMKS platform integration.

**wallet-asset-cleanup**, **wallet-outbox**, **wallet-validator**, **wallet-resilience-suite**,
**match-wallet-event-tracker** — wallet-family utilities.

**recurring-api** — recurring buy (DCA), CQRS + Event Sourcing (dormant repo).

**input-validator** (gRPC :50051, both clusters) — address/amount validation.

**platform** / **platform-mono** — umbrella repos aggregating services + infra for local full-system runs.

## ⚠️ Archived — don't send anyone there

`blackswan-gitops`, `blackswan-infrastructure`, `auth-service` (repo), `gateway`, `match-core`,
`revoke-relay`, `ws-server`, `notification-templates`, `mkk`, `web-archived`,
`fraud-aware-otp-service`, `order-event-forwarder`, `commission-cost-tracker`, `finance-dashboard`,
`v4-asset-*`, `wallet-match-event-tracker`, all `poc-*`.
