# transaction

> Deposit & withdraw operations for both crypto and fiat. Full lifecycle management.

- **Repo:** p-blackswan/transaction
- **Lang:** Go
- **Port:** gRPC **:50051**  (namespace **`shelby`**, not `blackswan`)
- **DB:** PostgreSQL

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
- Deployed as: transaction (command), transaction-query (read-mono), transaction-worker, transaction-job, transaction-ledger-outbox
- Related: **onchain** (p-blackswan, TypeScript) — blockchain monitoring
