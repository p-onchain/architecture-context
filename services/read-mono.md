# read-mono

> CQRS read-side monorepo. **28 domains**, **43 deployable apps in prod on AWS** + **6 more on
> Huawei**. Projections build read-optimized views from Kafka; query services serve gRPC. Not every
> domain has both.

- **Repo:** p-blackswan/read-mono
- **Lang:** Go — **per-service modules** (each `projection`/`query`/`worker` has its own `go.mod`).
  Versions differ per module (`1.25.3` / `1.25.4` / `1.26.1`). `go.work` is local-dev convenience
  only; CI/prod build each module independently.
- **Last verified:** 2026-08-17 (`.cd/helm/{prod,hw-prod,test}` + live prod pods)

## Structure

```
services/<domain>/{projection,query,worker,...}/   # each with its own go.mod
libs/                                              # older in-repo toolkit (see below)
.cd/helm/{prod,test,hw-prod,hw-test}/<app>.yaml    # ONE ArgoCD app per file
services/<domain>/CLAUDE.md                        # per-domain agent notes + traps — read these
```

## Domains (28)

alarm, anomaly-detection, balance, bank-integration, bank-integration-stock, campaign, commission,
cost-basis, custody-integration, **defi**, financial-history, klines, **notify**, open-order,
orderbook, orderbook-wapi, pass, pnl, pnl-agg, **portfolio**, **reward**, staking, ticker,
transaction, user, user-state, uservolume, ws

Recently changed:
- **new:** `defi`, `portfolio`, `reward`, `notify`, `bank-integration-stock`
- **removed:** `favourite`, `feedback` query services (decommissioned, E2B-97)
- `orderbook-wapi` is separate from `orderbook` — it serves the WAPI WebSocket orderbook stream with
  its own Redis key schema (centralized in the `rediskeys` package)

## Where each app runs

**exc-prod-alpha (AWS) — 43 apps.** Namespace comes from the chart value
`blackswan-service.namespace`, not from ArgoCD:

| Namespace | Apps |
|---|---|
| `saul` (29) | ticker-{projection,query,worker}, klines-{projection,query}, pnl-{projection,query,worker}, pnl-agg-query, cost-basis-{projection,query}, uservolume-{projection,query}, **defi-{projection,query}**, **portfolio-query**, orderbook-{projection,query}, orderbook-wapi-projection, open-order-{projection,query,reconciler}, financial-history-{projection,query}, commission-{api,daily-worker,migration-worker,process-worker,segment-projection} |
| `shelby` (6) | user-state-{projection,query}, campaign-{projection,query}, custody-integration-query, reward-query |
| `corleone` (3) | alarm-query, staking-query, ws-projection |
| `gopanel` (3) | anomaly-detection-{projection,query,worker} |
| `blackswan` (2) | balance-{projection,query} |

**exc-prod-hw (Huawei) — 6 apps:** `notify-projection`, `notify-query` (ns `saul`),
`user-query`, `transaction-query`, `bank-integration-query`, `reward-query`.
⇒ The **notification read side and the user/transaction read models exist only on Huawei in prod.**

**Test-only (not in prod yet):** `open-order-v2-{projection,query}`, `pass-projection`.

## Which Kafka does a service use?

Both. Pick from the service's helm values — never assume.

| Broker | read-mono consumers |
|---|---|
| **Internal Redpanda** (`10.240.*:9092`, plaintext) | `orderbook-projection`, `orderbook-wapi-projection` only |
| **AWS MSK** `excprodexternal` (SASL_SSL/SCRAM-SHA-512, user `msk_external_saul`) | every other Kafka-touching app: ticker, klines, pnl, cost-basis, uservolume, defi, balance, open-order, financial-history, campaign, user-state, anomaly-detection, commission-*, ws |
| — | all `*-query` services have **no Kafka at all** |

Schema Registry for all of them: `schema-registry-{0,1,2}.schema-registry.blackswan.svc.cluster.local:8081`.

⚠️ Trading topics reach MSK as **mirrored copies** written by `redpanda-connect`. MSK retention
therefore bounds any offset-reset rebuild or sorted replay.

### Consumed topics per projection (from `.cd/helm/prod`, 2026-08-17)

| Projection | Topics (env var prefix in parens) |
|---|---|
| `ticker-projection` | `order.events.match` (MATCH) + `orderbook.bestbidask` (ORDERBOOK) |
| `klines-projection` | `order.events.match` |
| `cost-basis-projection` | `order.events.match` |
| `uservolume-projection` | `order.events.match` |
| `open-order-projection` | `order.events.status,order.events.match` |
| `financial-history-projection` | `order.events.match,order.events.status,commission,transaction-events,`**`order.events.defi`** (the v1 DEX topic lives on only for this consumer) |
| `balance-projection` | `ledger-logs` |
| `pnl-projection` | `ledger-logs` (LEDGER) + `ticker.daily,ticker.last24h` (TICKERS) + `pnl.defi.asset.updates` (DEFI, `DEFI_CONSUMER_ENABLED=true`) + `pnl.user.asset.updates` (INTERNAL, self) |
| `defi-projection` | `order.events.dex` |
| `ws-projection` | `order.events.match,orderbook.bestbidask,commission` → produces `websocket-events` |
| `anomaly-detection-projection` | `ledger-logs` + `transaction-events` + `order.events.match` + **`auth-audit-events`** + `orderbook.bestbidask` |
| `user-state-projection` | `user-events` + `transaction-events` → produces `user-state-events` |
| `commission-segment-projection` | `user-events` |
| `campaign-projection` | `campaign-events` |
| `orderbook-projection`, `orderbook-wapi-projection` | `orderbook.state` — note the **`REDPANDA_`** env prefix, which is how you spot the Redpanda-direct consumers |

## Datastores (prod)

| Store | Domains |
|---|---|
| **ClickHouse Cloud** (3 separate services, via `*.eu-central-1.vpce.aws.clickhouse.cloud`) | `ticker`, `klines`, `uservolume` |
| **Postgres RDS** `exc-prod-<domain>` | balance (`balance_service`), pnl (`pnl_service`), order (`order_service` — open-order), financial-history (+ `-read-1` replica), commission, campaign, custody-integration, user-state, staking, reward, gopanel (`anomaly_detection_service`) |
| **Postgres, single DSN** | `defi` — `DEFI_DB_URL` from Vault (owns `dex_settlement_events`, `user_dex_holdings`) |
| **ElastiCache Redis** `master.exc-prod-<domain>` | ticker, pnl, orderbook, ws, user-volume, user-state, commission, financial-history, custody-integration, anomaly-detection |

## Key Details

- Standard toolkit is **corekit**. ⚠️ Pinned versions span `v0.21.3` → `v0.41.x` across modules — a
  corekit fix is not live until *that* module is bumped. Also `corekit/excluded` (excluded-user
  membership cache) and `corekit/userquery` (`GetBulkUsers`) are the shared primitives the read side
  uses to skip market-maker/bot accounts.
- In-repo `libs/` (messaging, cache, grpc, database, workerpool, partition, adapters) is an **older
  toolkit**, now used by only a handful of services (bank-integration, commission,
  custody-integration, staking) — prefer corekit.
- Protobuf from `proto-hub`; observability via `observer` (OTEL + Pyroscope).
- **Money uses `govalues/decimal`, NOT `fixedpoint`.** Some newer services also use
  `shopspring/decimal` — check per-service `go.mod`, and watch rounding parity between the two.
- Single generic Dockerfile for all services (`SERVICE`/`TYPE` build args).
- Projections are idempotent (DB-enforced `ON CONFLICT`) — rebuildable by resetting consumer offsets.
  ⚠️ Counter-style projections need an **event-identity** guard, not just an upsert, or redelivery
  double-counts.
- Deploys as `{domain}-projection` + `{domain}-query` (+ optional worker), Argo Rollouts blue-green.
  Some projections now autoscale with **KEDA** (financial-history first, E3B-101).

## The tightly-coupled core: ticker is the hub

```
                 match (order.events.match, via MSK mirror)
                   │
   config-service ─┤  orderbook domain (orderbook.bestbidask, internal)
   (precision,HTTP)│   │
                   ▼   ▼
                ┌─ ticker ─┐
   ticker.daily │          │ gRPC (TickerService / RateSource)
   ticker.last24h          ├───────────┬───────────┐
   (Kafka)      │          │           │           │
                ▼          ▼           ▼           ▼
              pnl      uservolume  cost-basis    pnl-agg
```

- **pnl** ← ticker (`ticker.daily`, `ticker.last24h` — **two producers**: read-mono ticker-worker for
  CEX prices and onchain for DeFi tokens) + `ledger-logs` (wallet) + `pnl.defi.asset.updates` (defi)
- **uservolume** → ticker (gRPC `TICKER_ADDRESS`)
- **cost-basis** → ticker (gRPC `RateSource`; the client file is misleadingly named
  `uservolume_client.go` but calls `tickerv1.TickerServiceClient`). **CEX-only** — no DEX, no producer.
- **pnl-agg** — query-only aggregator, no Kafka. Fans out to ticker (`TICKER_SERVICE_URL`, DeFi prices
  behind `GetTicker include_defi=true`) plus balance/pnl/klines/staking/transaction/cost-basis/
  defi-pnl query services and **defi `GetUserHoldings`**
- **defi** ← onchain `order.events.dex` (v2). Owns the DEX holdings/cost/realized-PnL read model
  (rebuilt by **sorted replay**), **produces** `pnl.defi.asset.updates`, serves
  `defi.query.v1.DefiQueryService.GetUserHoldings` to pnl-agg. **Live in prod** — the DEX CQRS
  migration completed 2026-08-01 (onchain#241 removed the legacy `/v1/onchain/accounts/pnl*` path).

⇒ Touching ticker's proto/events/gRPC means regression-checking pnl, uservolume, cost-basis and pnl-agg.

## Operational traps (things the code won't tell you)

- **CI proves almost nothing here:** the `golangci-lint` step is a permanent no-op and there is no
  `go test` job. Build/test each touched module locally.
- **Pushes touching `.github/**` are rejected wholesale** by a branch-protection ruleset with no
  bypass actors — split those edits out.
- After branching, `git log origin/main..HEAD` — a branch cut can sweep in someone else's main commit.
- `revive`'s `unused-parameter`: blanked params must be bare `_`, not `_name` (`unparam` skips
  exported methods, so revive is the only linter that fires).
- ClickHouse Cloud **autoscaling** is a recurring incident source for ticker/klines (night-time
  downscales, generation swaps that look like failures). When comparing metrics across a swap use
  `avg`, not `sum(rate(...))`, and never sum `avg_over_time` across pod generations — dead
  blue-green pods count as concurrent.
- Redis/ElastiCache **egress** can be clamped while engine latency looks healthy — the tell is
  variance collapse in `redis_net_output_bytes_total`, not latency.
- Storage growth is a live concern (cost-basis `cex_match_events`, pnl `event_dedup`): RDS storage
  never shrinks, and a table rewrite needs more free space than a nearly-full disk has. Retention
  before reshape.
- Client-cancelled gRPC calls are **censored from server histograms** — an absent latency spike does
  not mean the RPC was fine.

## Related docs

- `INDEX.md` — clusters, namespaces, brokers, topics
- `conventions.md` — module layout, ticket/branch rules, deploy mechanism
- `flows/order-lifecycle.md` — where the projections sit in the trade path
- Per-domain `services/<domain>/CLAUDE.md` inside the repo — the real trap list
