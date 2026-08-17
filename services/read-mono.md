# read-mono

> CQRS read-side monorepo. 25+ service domains (~15 projections + ~21 query services + a few workers; ~43 deployable units in prod) that build read-optimized views from Kafka events. Not every domain has both a projection and a query.

- **Repo:** p-blackswan/read-mono
- **Lang:** Go 1.26+ — per-service modules (each `projection`/`query`/`worker` has its own `go.mod`); `go.work` is local-dev convenience only, not used in CI/prod
- **DB:** PostgreSQL (per-domain), ClickHouse (analytics: klines/ticker/uservolume), Redis (orderbook snapshots, klines/ticker cache, commission state)

## Structure

Most domains have:
- **projection** — Kafka consumer → writes to DB/cache (no API)
- **query** — gRPC server → serves reads to BFF (no Kafka)

Some domains also add a **worker** (e.g. ticker, pnl, commission); a few are projection-only (e.g. open-order).

## Domains

alarm, anomaly-detection, balance, bank-integration, campaign, commission, cost-basis, custody-integration, defi, favourite, feedback, financial-history, klines, open-order, orderbook, orderbook-wapi, pass, pnl, pnl-agg, staking, ticker, transaction, user, user-state, uservolume, ws

> `orderbook-wapi` is a separate domain from `orderbook` — it serves the WAPI WebSocket orderbook stream with its own Redis key schema (centralized in `rediskeys` package).

> **`defi`** is the DEX/on-chain read side (CQRS migration of DEX accounting out of the `onchain` service). It consumes `order.events.dex` (v2, produced by `onchain`), owns DEX holdings/cost-basis/realized-PnL (own Postgres `defi` DB, rebuilt by sorted replay), **produces** `pnl.defi.asset.updates` (→ `pnl`), and serves `defi.query.v1.DefiQueryService.GetUserHoldings` over gRPC (→ `pnl-agg`). As a result **`cost-basis` is now CEX-only** (it no longer tracks DEX). The `defi` service is merged + live on `phoenix-test`; the `ticker` DeFi-price feed and the `pnl-agg` re-point onto `defi-query` are still in flight, and prod is a launch (not yet live).

## Key Details

- Standard toolkit is **corekit** (`p-blackswan/corekit`) — Kafka (consumer + batch/simple processors + DLQ, producers), DB (postgres/redis/clickhouse), gRPC, envcfg, xlogger. Used by nearly all service modules.
- In-repo `libs/` (messaging, cache, grpc, database, workerpool, partition, adapters) is an **older toolkit**, now used by only a handful of services (bank-integration, commission, custody-integration, favourite, feedback, staking) — prefer corekit.
- Protobuf from `proto-hub`; observability via `observer` (OTEL + Pyroscope).
- **Money uses `govalues/decimal`, NOT the fleet's `fixedpoint`.** Don't introduce `fixedpoint` in this repo. Newer services also use `shopspring/decimal` — check per-service go.mod.
- Single generic Dockerfile for all services (`SERVICE`/`TYPE` build args)
- Projections are idempotent (DB-enforced via `ON CONFLICT`) — can rebuild by resetting consumer offsets
- Each domain deploys as `{domain}-projection` + `{domain}-query` (+ optional `{domain}-worker`) pods

## Performance Notes (2026-05+)

- **GOMAXPROCS aligned to CPU limit** — all services now set GOMAXPROCS to match container CPU limit, not runtime default
- **Redis pool sized to GOMAXPROCS** — pool size set explicitly to match GOMAXPROCS (not library default) to avoid connection contention
- **pnl-projection batched** — switched to batch processor to drain ledger lag faster (EXCH-3309)
- **ledger-path rule evaluation parallelized** — Redis round-trips cut by parallelizing rule evaluation across rules (EXCH-0000, 2026-06)
- **balance-projection batch-tuned** — batch processor config tuned for throughput (EXC-1895)
- Memory requests/limits raised to **1Gi** for projection pods (2026-06)

## Bug Fixes & Behavioral Notes (2026-06)

- **daily PnL baseline** — positions opened today now use `avg` cost as baseline instead of 0 (EXCH-0000; prevented intraday PnL from being overstated for same-day opens)
- **price_deviation persist_window** — new tunable `persist_window` suppresses short flicker alerts; reports the full deviation duration, not just point-in-time (EXCH-5351)
