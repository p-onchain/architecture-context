# bff-api

> Backend-for-Frontend for the web client. Aggregates gRPC microservices into HTTP/JSON.

- **Repo:** p-blackswan/bff-api
- **Lang:** Go
- **Port:** HTTP :3000 · namespace `blackswan` · internal route `bff-api.int.paribu.com`
- **Stateless proxy** — no DB (Redis only, for MFA session state)
- **Its client is `micro-web`** (the `paribu-*` micro-frontends behind `paribu-orchestrator`), not the
  retired `web` SPA. See `clients/web.md`.
- **Last verified 2026-08-17** from `.cd/helm/prod/`.

## Talks To

Behind KrakenD; fans out over gRPC. The address form tells you which cluster the callee is in:

**Same cluster (AWS `exc-prod-alpha`) — in-cluster DNS `<svc>.<ns>.svc.cluster.local:50051`:**

| Namespace | Upstreams |
|---|---|
| `saul` | ticker-query, klines-query, orderbook-query, open-order-query, financial-history-query, uservolume-query, pnl-query, commission-api, notify-api (HTTP :8080) |
| `blackswan` | order-api (HTTP :3000), conditional-order, order-strategies, balance-query, krakend-revoke-server (HTTP :8081) |
| `shelby` | auth-service (mellon, HTTP :8080 — env vars still say `MONOSIGN_*`), config-service (HTTP :8080), input-validator, user-state-query |
| `corleone` | alarms-server (**:50082**), alarm-query |
| `saul`/other | global-price-tracker-api |

**Cross-cloud (Huawei `exc-prod-hw`) — via that cluster's internal gateway,
`<svc>.internal.paribu.com:50051`:** `user` (user-service), `user-query`, `transaction`,
`transaction-query`, `notify-query`, `support-api`.

> **Hostname rule worth memorising:** `*.int.paribu.com` = AWS internal gateway,
> `*.internal.paribu.com` = Huawei internal gateway. A dependency addressed by hostname rather than
> cluster DNS is a **cross-cloud hop** — latency and failure modes differ.

Public auth host: `https://account.paribu.com` (`MONOSIGN_BASE_URL`).

## Key Details

- No Kafka — purely a proxy/aggregation layer
- Feature flags via Flipt v2; profiling via Grafana **Alloy** (`alloy.monitoring.svc.cluster.local:4040`)
- bff-client is the same pattern for mobile (Samaritan); bff-instant-pay is a third, fiat-focused BFF

## Notable Behaviors

- **Orderbook gating (E4B-30):** the orderbook endpoint checks market live status before serving.
  Pre-launch markets return 403 unless the user has `pre_launch_access=true` in their config-service
  merged config — a per-request lookup, no caching. `marketstatus.Checker` is injected into
  `MarketHandler` alongside a `configadapter.Client`.
- **cost-basis dual-auth (EXCH-6876):** `/conversion/cost-basis` requires **both** a user JWT and a
  valid `X-Internal-Token`; KrakenD forwards both, bff-api validates both.
- **Notification inbox repoint (E1B-83):** the web `/v1/notification/*` path now goes through
  bff-client rather than bff-api's own wiring. bff-api's dark-launch side of that change is done —
  the flip landed with bff-client v1.3.13 on 2026-08-17 (+33–41 ms measured).
