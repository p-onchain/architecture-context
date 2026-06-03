# read-mono

> CQRS read-side monorepo. ~25 projection/query service pairs that build read-optimized views from Kafka events.

- **Repo:** p-blackswan/read-mono
- **Lang:** Go (monorepo with `go.work`)
- **DB:** PostgreSQL (per-domain), ClickHouse (analytics)

## Structure

Each domain has:
- **projection** — Kafka consumer → writes to DB
- **query** — gRPC server → serves reads to BFF

## Domains

alarm, anomaly-detection, balance, bank-integration, campaign, commission, cost-basis, custody-integration, favourite, feedback, financial-history, klines, open-order, orderbook, orderbook-wapi, pass, pnl, pnl-agg, staking, ticker, transaction, user, user-state, uservolume, ws

## Key Details

- Shared libs in `libs/` (messaging, cache, grpc, database, workerpool, partition, adapters)
- Single generic Dockerfile for all services
- Projections are idempotent — can rebuild by resetting consumer offsets
- Each domain deploys as `{domain}-projection` + `{domain}-query` pods
- Uses corekit for Kafka, DB, gRPC setup
