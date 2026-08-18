# bff-client

> Backend-for-Frontend for the mobile app (Samaritan). Same pattern as bff-api but with mobile
> response shapes — and a **much wider** surface: it now also fronts the BIST/US **equities** product
> and the helpdesk stack.

- **Repo:** p-blackswan/bff-client
- **Lang:** Go
- **Port:** HTTP :8080 · namespace `blackswan` · internal route `bff-client.int.paribu.com`
- **Stateless** — no DB
- **Last verified 2026-08-17** from `.cd/helm/prod/`.

## Talks To

Everything bff-api does (see `bff-api.md` for the cluster-DNS vs `*.int`/`*.internal` hostname rule),
plus these mobile-only / newer upstreams:

| Upstream | Address | Note |
|---|---|---|
| **defi-query** | `defi-query.saul…:50051` | DEX holdings — the read-mono `defi` domain |
| **pnl-agg-query** | `pnl-agg-query.saul…:50051` | portfolio aggregate |
| **portfolio-query** | `portfolio-query.saul…:50051` | Cüzdan / wallet read aggregate |
| **defi-api** (onchain) | `defi-api.onchain…:8080` | `ONCHAIN_API_URL` |
| **heimdall** | `dns:///heimdall-headless.momentum…:50051` | external data hub (headless → client-side LB) |
| **helpdesk-backend** | `helpdesk-backend.gopanel…:8080` | still called `TICKBU_API_URL` |
| **helpdesk-assistant** | `helpdesk-assistant.gopanel…:8090` | AI support assistant |
| **campaign / staking** | `campaign-service`, `campaign-query` (shelby), `staking-service`, `staking-query` (corleone) | command + query pairs |
| **order-strategies** | `order-strategies.blackswan…:50051` | `SERVICE_STRATEGY_API_URL` |
| **feedback** | `user.internal.paribu.com:50051` | back on user-service — read-mono's `feedback-query` was decommissioned |
| **KYC** | `http://kyc.internal.paribu.com` | `PKYC_API_URL`, Huawei-side (estel) |
| **bank-integration-query, transaction(-query), user(-query), notify-query, support-api** | `*.internal.paribu.com` | **cross-cloud hops to `exc-prod-hw`** |
| **Equities (xprstock)** | `https://{asset,bist-market-data,us-market-data,watchlist,alert}-api.prod.xprstock.run` | external BIST/US equities platform |
| **News** | `https://terminal-api.paribu.one` | |
| **CDN** | `https://cdn.paribu.com` | |

## Kafka

⚠️ Unlike bff-api, bff-client **does** use Kafka: it produces **`marketdata.access.events`** to MSK
(`msk_external_blackswan`) from `internal/adapters/bistaccessevent/` — the BIST market-data access
trail. Schema Registry is the usual Confluent SR in ns `blackswan`.

## Key Details

- Sits behind KrakenD; feature flags via Flipt v2; profiling via Grafana Alloy
- Uses corekit + proto-hub
- ⚠️ **A panic in any of its ~230 errgroup goroutines takes down the whole BFF** —
  `platform.Recovery` only wraps the HTTP handler and there is no `SafeGo` helper. Treat new
  `errgroup.Go` calls in fan-out paths as a reliability hazard.
- ⚠️ Mobile users see a generic **"Bir şeyler ters gitti" dialog** for a bff 503 (gRPC `Unavailable`
  → 503). That popup is *not* an app crash — client-side errors land in Sentry, not here.

## Notable Endpoints & Behaviors

- **GET /v1/banners** — hardcoded home banners (no downstream call); shapes owned in `internal/feature/banner/`
- **GET /v1/onchain/health** — public health probe, IP rate-limited at the gateway (60 req/min)
- **POST /v1/onchain/…/challenge** — gateway forwards `X-Forwarded-For` for audit
- **Helpdesk attachment upload** — 25 MiB body cap (bff-client's `http.MaxBytesReader` is authoritative),
  60s timeout; gateway rejects chunked bodies; ClamAV limit 30 MiB gates any increase
- **Notification inbox (E1B-83):** bff-client now serves `/v1/notification/*` for **both** web and
  mobile — the web repoint went live with v1.3.13 on 2026-08-17
- **Legacy DEX P&L proxies are gone:** the old `/v1/onchain/accounts/pnl*` passthrough was removed with
  the DEX CQRS cutover (onchain#241, prod 2026-08-01); DEX P&L now comes via pnl-agg / defi-query
- **Portfolio:** staking APY per portfolio item; loyalty balance aligned with portfolio total
- **Alarm history:** includes `type` and full `caip19` identifier
- **KYC video upload:** `SERVER_READ/WRITE_TIMEOUT` 120s + graceful-shutdown drain to avoid mid-upload kills
- ⚠️ Sub-cent assets once rendered as ₺0 because `asset_detail.go` formatted with `StringFixed(2)` —
  formatting precision is a live class of bug here, not a data problem
