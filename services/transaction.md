# transaction

> Deposit & withdraw operations for both crypto and fiat. Full lifecycle management.

> **Cluster placement verified 2026-08-17**; the module/flow detail below dates from 2026-06.

- **Repo:** p-blackswan/transaction
- **Lang:** Go
- **Port:** gRPC **:50051**  (namespace **`shelby`**, not `blackswan`)
- **⚠️ Runs on Huawei Cloud in prod** — cluster **`exc-prod-hw`**, not `exc-prod-alpha`. Values live in
  `.cd/helm/hw-prod/`. Same for `transaction-worker`, `transaction-job`,
  `transaction-ledger-outbox` and read-mono's `transaction-query`. (`legacy-custody-bff`, also from
  this repo, runs on AWS.)
- **DB:** PostgreSQL
- **Kafka:** MSK — produces `transaction-events`, `address-events`

## Talks To

- **wallet** (gRPC) — balance mutations (add/remove/withdraw/finalize)
- **custody-integration** (gRPC) — HSM/custody signing for crypto
- **bank-integration** (gRPC) — bank API for fiat
- **user-service** (gRPC) — user lookups
- **elliptic-screener** — AML screening
- **block-listener** — blockchain deposit notifications

## Kafka

- Produces: transaction lifecycle events (deposit created/verified, withdraw created/signed/verified, etc.)
- All event protos in `proto/transaction/event/v1/`

## Key Details

- Large service with many internal modules (fraud, staking rewards, travel rule, treasury balance, suspicious tx watcher, etc.) — clone the repo to explore
- Deployed as: transaction (command), transaction-query (read-mono), transaction-worker, transaction-job, transaction-ledger-outbox — **all on `exc-prod-hw`**
- Fiat neighbours on the same cluster: bank-integration(+query), invoice-service, reconciliation,
  pdf-operations, soap-instant-pay, bff-instant-pay
- Related: **onchain** (p-blackswan, TypeScript/Bun) — deposit detection *and* the DEX/WaaS platform;
  see `services/onchain.md`. Note the transaction service is a *notify fire-and-forget* caller: a
  notify 503 silently loses the notification rather than back-pressuring.
