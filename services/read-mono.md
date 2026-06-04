# read-mono

> CQRS read-side monorepo. ~25 projection/query service pairs that build read-optimized views from Kafka events.

- **Repo:** p-blackswan/read-mono
- **Lang:** Go — per-service modules (each `projection`/`query`/`worker` has its own `go.mod`); `go.work` is local-dev convenience only, not used in CI/prod
- **DB:** PostgreSQL (per-domain), ClickHouse (analytics: klines/ticker/uservolume), Redis (orderbook snapshots, klines/ticker cache, commission state)

## Structure

Most domains have:
- **projection** — Kafka consumer → writes to DB/cache (no API)
- **query** — gRPC server → serves reads to BFF (no Kafka)

Some domains also add a **worker** (e.g. ticker, pnl, commission); a few are projection-only (e.g. open-order).

## Domains

alarm, anomaly-detection, balance, bank-integration, campaign, commission, cost-basis, custody-integration, favourite, feedback, financial-history, klines, open-order, orderbook, orderbook-wapi, pass, pnl, pnl-agg, staking, ticker, transaction, user, user-state, uservolume, ws

## Key Details

- Standard toolkit is **corekit** (`p-blackswan/corekit`) — Kafka (consumer + batch/simple processors + DLQ, producers), DB (postgres/redis/clickhouse), gRPC, envcfg, xlogger. Used by nearly all service modules.
- In-repo `libs/` (messaging, cache, grpc, database, workerpool, partition, adapters) is an **older toolkit**, now used by only a handful of services (bank-integration, commission, custody-integration, favourite, feedback, staking) — prefer corekit.
- Protobuf from `proto-hub`; observability via `observer` (OTEL + Pyroscope).
- **Money uses `govalues/decimal`, NOT the fleet's `fixedpoint`.** Don't introduce `fixedpoint` in this repo.
- Single generic Dockerfile for all services (`SERVICE`/`TYPE` build args)
- Projections are idempotent (DB-enforced via `ON CONFLICT`) — can rebuild by resetting consumer offsets
- Each domain deploys as `{domain}-projection` + `{domain}-query` (+ optional `{domain}-worker`) pods
