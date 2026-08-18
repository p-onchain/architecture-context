# BlackSwan Exchange — Architecture Context

> Lightweight service map for AI coding agents. Tells you WHERE to find things, not HOW they work internally — clone the repo and read the code for that.

**Last verified: 2026-08-17** against live prod (ArgoCD app catalog + VictoriaMetrics `kube_pod_info`
via sre-mcp), `p-platform/platform-gitops@main`, and per-repo `.cd/helm/<env>/*.yaml`. Anything
dated 2026-06 or earlier below survived that check; claims that did not are marked ⚠️.

## System Overview

BlackSwan is a crypto exchange platform (Paribu v6). Go microservices, CQRS/Event Sourcing,
gRPC (sync) + Kafka (async). Kubernetes via ArgoCD GitOps — **multi-cloud: AWS EKS for the
exchange core and read side, Huawei Cloud CCE for the user/transaction/finance tier.**

## Clusters & Namespaces (prod)

The exchange runs in **two** prod clusters. Which cluster a service lives in is not guessable
from its repo — check `.cd/helm/{prod,hw-prod}/` in the repo, or the table below.

| Cluster | Cloud | Namespaces (pods) |
|---|---|---|
| **exc-prod-alpha** | AWS EKS | `blackswan` ~572 · `saul` ~201 · `gopanel` ~48 · `momentum` ~36 · `corleone` ~26 · `onchain` ~20 · `shelby` ~49 |
| **exc-prod-hw** | Huawei CCE | `shelby` ~43 · `saul` 5 · `blackswan` 4 · `corleone` 2 |

| Namespace | What's in it |
|---|---|
| `blackswan` | Trading hot path — **263 `match-{currency}-{payment}` engines**, order-api, order-responder (~64 pods), order-strategies, wallet, wallet-ledger-sink, wallet-validator, balance-projection/query, match-forge, conditional-order, market-api, krakend-gateway, bff-api, bff-client, commission-update-worker, **schema-registry** (3, Confluent/CFK), **redpanda-connect** (~64), flipt-v2, temp-enqura-proxy |
| `saul` | CQRS **read side** — 29 read-mono apps (ticker, klines, pnl, pnl-agg, cost-basis, uservolume, defi, portfolio, orderbook, open-order, financial-history, commission, anomaly-detection…) + notify-api (notif2) + global-price-tracker |
| `shelby` | Accounts / financial. **Split across clouds:** AWS → auth-service (mellon), config-service(+worker), campaign-*, custody-integration(+query), input-validator, kep, legacy-custody-bff, user-state-projection/query, bff-instant-pay. Huawei → **user-service** (+worker/jobs/query, tier-worker), **transaction** (+worker/job/ledger-outbox/query), bank-integration(+query), invoice-service, reconciliation, reward-service (+consumer/distributor/jobs/query), sanctions, pdf-operations, soap-instant-pay |
| `corleone` | Alarms (alarms-server, quick-/custom-alarms-worker, alarm-query), staking-{service,query,distributor,jobs}, **wapi**, **ws-hub**, ws-projection, partner-integration (HW) |
| `onchain` | DeFi / WaaS — defi-api, defi-magic-spend, 5 ERC-4337 bundlers (base, bsc, polygon, hyperevm, **robinhood**), defi-hyperliquid-stream, defi-polymarket-stream, defi-scheduler-{critical,periodic} |
| `gopanel` | Back-office & analytics — gopanel-backend/frontend, crm-* (5), dash-* (4), helpdesk-{backend,frontend,assistant}, support-web, webhelp, anomaly-detection read-mono apps, clamav |
| `momentum` | heimdall (Rust) + the **12 micro-web frontends** (`paribu-orchestrator`, header, footer, home, home-ssr, lssr, market-detail, terminal, wallet, account, auth, overlay) |
| `saul` (HW only) | read-mono **notify-projection / notify-query** — the notification read side runs on Huawei ONLY |

⚠️ **Trap:** sre-mcp's `sre_list_services` reports `namespace: exchange` for ~140 prod apps.
That is the ArgoCD Application *destination default*, not where pods run — the Helm chart sets
its own `blackswan-service.namespace`. There is no `exchange` namespace with pods. Trust
`kube_pod_info` / the values file, not the ArgoCD app metadata.

⚠️ An ArgoCD app existing ≠ a workload running. `applicationsSync: create-update` +
`preserveResourcesOnDeletion` means apps are never pruned, so orphans persist (e.g. `mkk` has an
app and an **archived** repo but zero pods). CronJob-shaped apps (`log-timestamp-*`,
`eventificator-backfill`) also show no steady-state pods.

## Request Flow

```
Client (micro-web / Samaritan / WAPI)
  → Cloudflare (TLS, WAF, DDoS — no log access via sre-mcp)
    → Envoy Gateway (Gateway API, ns gateway-api-system; NLB + ACM)
       ├─ external-gw     — public traffic
       ├─ internal-gw     — in-VPC service exposure (HTTPRoute + GRPCRoute)
       └─ internal-ops-gw — ops UIs
      → KrakenD (service.name=`krand` in SigNoz; :8080) — JWT / API-key auth, rate limiting
        → BFF (bff-api :3000 web · bff-client :8080 mobile · bff-instant-pay)
          → gRPC microservices (almost all :50051)
            → Kafka (async) → read-mono projections → query services → back to BFF
```

**Public hostnames** (`external-gw`):

| Hostname | Backend |
|---|---|
| `web.paribu.com`, `api.paribu.com`, `app.paribu.com`, `ext.paribu.com`, `mobile.paribu.services`, `*-v6.paribu.com` | krakend-gateway:8080 |
| `paribu.com`, `www.paribu.com` | **paribu-orchestrator:8080** (micro-web); `/blog` → support-external, `/hub` → hub-external |
| `go-v6.ceteris.io` | gopanel-frontend / -backend |
| `schema-registry.paribu.com` | schema-registry (basic-auth SecurityPolicy) |

**Internal exposure** — `internal-gw` publishes ~58 `<service>.int.paribu.com` names, gRPC via
**GRPCRoute** on `sectionName: grpc` (e.g. `ticker-query.int.paribu.com`, `pnl.int.paribu.com`,
`defi-query.int.paribu.com`, `user-state-query.int.paribu.com`). Reachable over VPN — useful for
per-user prod lookups with `grpcurl` (reflection is on, plaintext).
`internal-ops-gw` serves ops UIs: `grafana.int.paribu.com`, `vmalert.int.paribu.com`,
`vault.int.paribu.com`, `kafka.int.paribu.com` (kafka-ui), `msk.int.paribu.com` (redpanda-msk-console),
`schema.int.paribu.com`, `notification-ops.int.paribu.com`.
Definitions: `platform-gitops/applications/http-routes/values/<cluster>.yaml`.

## Messaging — TWO brokers plus a bridge

⚠️ The old "Redpanda internally, MSK only as a legacy external bridge" model is **wrong**.

| Bus | Endpoint | Auth | Who uses it |
|---|---|---|---|
| **Internal Redpanda** | `10.240.65.47,10.240.130.74,10.240.252.225,10.240.104.25,10.240.178.238` : **9092** (5 brokers, plaintext, in-VPC, not in K8s) | none | The latency-critical path only: match, order-api, order-responder, wallet, conditional-order, wapi, and read-mono `orderbook-projection` + `orderbook-wapi-projection` |
| **AWS MSK** `excprodexternal` | `b-{1,2,3}.excprodexternal.8dbck2.c6.kafka.eu-central-1.amazonaws.com:9096` | SASL_SSL / SCRAM-SHA-512, per-namespace users (`msk_external_saul`, `msk_external_blackswan`) | Everything else — all other read-mono projections, notif2, onchain, user-service, mellon, transaction, config-service, crm-service, staking, alarm-service, ws-hub, bff-client |

**`redpanda-connect`** (Benthos, ~64 pods, ns `blackswan`) bridges them both ways, preserving
topic names. Config: `platform-gitops/applications/redpanda-connect/values/<cluster>.yaml`.

- Redpanda → MSK (raw proto): `order.events.match`, `order.events.status`, `ledger-logs`, `wallet-error-events`, `orderbook.analysis.snapshot`
- Redpanda `ledger-logs` → decode / extract `.commission` / re-encode → MSK topic **`commission`** (subject `commission-value`)
- MSK → Redpanda (raw proto): `user-commission-events`, `config-service-events`, `user-state-events`

So a read-side consumer of a trading topic reads a **mirrored copy on MSK** — MSK retention, not
Redpanda retention, bounds any replay/rebuild.

**Schema Registry:** one self-hosted **Confluent** SR (`cp-schema-registry` 8.0.3, 3 replicas,
managed by the CFK operator) in ns `blackswan`, storing its schemas in MSK. In-cluster:
`schema-registry-{0,1,2}.schema-registry.blackswan.svc.cluster.local:8081`; external:
`https://schema-registry.int.paribu.com`. Services set `AUTO_REGISTER=false`,
`USE_LATEST_VERSION=true` (schemas pre-registered from proto-hub CI).

## Key Kafka Topics

| Topic | Producer → Consumers |
|---|---|
| `order.events.match` | match → wallet, order-responder, match-forge, wapi, mkk; **(via MSK mirror)** read-mono klines/ticker/uservolume/cost-basis/open-order/financial-history, ws-projection |
| `order.events.status` | match, conditional-order → order-responder, wallet, match-forge, conditional-order, wapi; read-mono open-order/financial-history |
| `orderbook.match_price` | match → order-api, conditional-order, match-forge, wapi |
| `orderbook.state` | match → match-forge, read-mono orderbook-projection |
| `orderbook.bestbidask[.*]` | read-mono orderbook-projection → ticker-projection, ws-projection |
| `orderbook.analysis.snapshot` | orderbook side → mirrored to MSK (dash / anomaly-detection) |
| `order.requests.{market}` | match WAL (per-market suffixed) → match replays its own WAL on boot; match-forge tails `order.requests.*` |
| `ledger-logs` | wallet → wallet-ledger-sink, balance-projection, pnl-projection, mkk, wapi |
| `commission` | redpanda-connect (extracted from ledger-logs) → ws-projection, financial-history-projection |
| `user-commission-events` | commission-process-worker → wallet, commission-update-worker |
| `config-service-events` | config-service(+worker) → order-api, wallet, match |
| `user-events` | user-service, user-worker → user-state-projection, commission-segment-projection, quick-alarms-worker, mkk |
| `user-state-events` | user-state-projection → order-api (`CanTrade`) |
| `transaction-events` | transaction(+worker/job) → user-state-projection, financial-history-projection, mkk |
| `ticker.daily`, `ticker.last24h` | **two producers:** read-mono ticker-worker (CEX) + onchain (DeFi tokens) → pnl-projection, quick-alarms-worker |
| `pnl.user.asset.updates` | pnl-projection → pnl-projection (self) |
| `order.events.dex` (v2, `proto.dex.order.v2`) | onchain → read-mono **defi**-projection |
| `pnl.defi.asset.updates` | read-mono **defi** → read-mono **pnl** (sole producer post-cutover) |
| `order.events.defi` (v1) | legacy, idle — retained for financial-history |
| `websocket-events` | balance-projection, ws-projection, config-service, transaction, financial-history-projection, orderbook-projection, ticker-worker → ws-hub |
| `auth-audit-events` | mellon/auth-service → **anomaly-detection-projection** |
| `campaign-events` | campaign-service → campaign-projection |
| `notification.request`, `notification.request.bulk`, `notification.news.push` | callers (`corekit/notify`) / CRM → notif2 `notify-api` |
| `marketdata.access.events` | **bff-client** → equities entitlement trail |
| `order.events.hyperliquid`, `order.events.polymarket`, `hyperliquid-{ledger,position}-events`, `defi-audit-events` | onchain → perps / prediction-market domains |
| `address-events`, `elliptic-screener-events`, `staking-events`, `partner-integration-events`, `alarm.worker.alarm-events`, `wallet-error-events`, `archive.transaction-events` | domain-local |

> Match/status/orderbook topics are **shared and un-suffixed** — consumers filter by market from
> the payload. Only the WAL (`order.requests.{market}`) is per-market suffixed.

> **DEX read side: 🏁 migration complete.** DEX accounting now lives in the read-mono `defi`
> domain (live in prod `saul`); onchain#241 removed the legacy `/v1/onchain/accounts/pnl*` path in
> prod on 2026-08-01. `cost-basis` is CEX-only. See `services/read-mono.md`.

## Service Map

| Domain | Services | Repos |
|---|---|---|
| **Trading** | order entry, matching, response aggregation, conditional triggers, strategies, event store | order-api, match, order-responder, conditional-order, order-strategies, match-forge, market-api, market-configs |
| **Wallet** | balances, ledger persistence, validation, cleanup | wallet, wallet-ledger-sink, wallet-validator, wallet-outbox, wallet-asset-cleanup |
| **Transaction** | crypto/fiat deposit & withdraw | transaction, bank-integration, custody-integration, legacy-custody-bff, bff-instant-pay, SoapInstantPay |
| **User & Auth** | users, KYC, auth (OIDC/OAuth2/passkeys/SSO) | user-service, **mellon** (deploys as `auth-service`), **estel** (Rust — kyc-api/-orchestrator/-backoffice), p-utilities/p-kyc |
| **Market Data** | external prices, market metadata, external data hub | global-price-tracker, market-api, heimdall (Rust) |
| **Notification** | push/SMS/email/in-app | **notif2** (`notify-api`, `notify-iys-worker`) + read-mono `notify` domain; notification (v1, being retired) |
| **Read Models** | 28 CQRS domains, 43 prod apps | **read-mono** (monorepo) |
| **Gateway / Edge** | Envoy Gateway, API gateway, BFFs | platform-gitops (gateway-resources, http-routes), krakend-gateway (+apikey/requestlog plugins), bff-api, bff-client, bff-instant-pay |
| **Real-time** | WebSocket | ws-hub (web/mobile JWT), wapi (API-key traders, direct Redpanda) |
| **DeFi / WaaS** | DEX trading, smart accounts, bundlers | onchain, bundler-erc4337, p-onchain/{p-accounts, p-accounts-sol, MagicSpend, kernel, 7579-plugins, swig-wallet, legolas, token-list} |
| **Admin / Support** | back-office, CRM, analytics, helpdesk | gopanel, crm-service, dash, helpdesk-paribu, assistant-paribu, support-web, webhelp, p-backoffice/paribu-one-* |
| **Compliance** | AML, sanctions, reporting | elliptic-screener, sanctions, mkk, paribu-one-compliance, kep-service, audit-log-api |
| **Finance** | invoicing, reconciliation, commissions, rewards | invoice-service, reconciliation, commission-update-worker, commission-transfer-job, reward-service |
| **Staking / Campaign** | staking pools, promotions | staking-service, campaign-service |
| **Config & Flags** | dynamic config, feature flags | config-service, **flipt-v2** + feature-flags (flag state) + feature-flags-go, ff-dash |
| **Events / Data** | archival, mirroring, backfill | eventificator, log-timestamp, redpanda-connect |
| **Infra / Platform** | GitOps, Helm base, ansible, catalog, SRE | **p-platform/platform-gitops**, blackswan-helm-base, p-platform/{ansible, service-catalog, sre-mcp}, kyc-ansible |

## Clients

| Client | Repo | Stack | Reaches |
|---|---|---|---|
| **Web** | p-blackswan/**micro-web** | Vue micro-frontends (12 apps, ns `momentum`), shell = `paribu-orchestrator` | bff-api via KrakenD; serves `paribu.com` |
| **Samaritan** (mobile) | pikachu-exchange/samaritan | React Native / Expo | bff-client via KrakenD; `mobile.paribu.services` |
| **WAPI** (trading API) | p-blackswan/wapi | Go WebSocket | direct internal Redpanda; API-key HMAC at KrakenD |
| **GoPanel** (admin) | p-blackswan/gopanel | Go + Vue | direct |
| **Paribu One** (back-office suite) | p-backoffice/paribu-one + `paribu-one-*` domain repos | Go + framework submodule | per-domain |

⚠️ `p-blackswan/web` (the old Vue SPA) is **no longer the prod web frontend** — `micro-web` is.
`web` still has a `.cd/helm/prod/values.yaml` but no running ArgoCD app; `web-archived` exists.

## Shared Libraries

| Library | Repo | What it gives you |
|---|---|---|
| **corekit** | p-blackswan/corekit | DB (postgres/redis/clickhouse), Kafka (consumer/processors/DLQ/producers), gRPC, envcfg, xlogger, logctx, apperrors, worker, xjobs, retry, httptransport, masking, clock, xsync, xtime, **excluded**, **notify**, **userquery**, testing. Current tag **v0.41.0** |
| **proto-hub** | p-blackswan/proto-hub | all protobuf definitions + generated Go |
| **observer** | p-blackswan/observer | OTEL exporters, Pyroscope, metrics/tracing |
| **blackswan-helm-base** | p-blackswan/blackswan-helm-base | shared Helm base chart (`blackswan-service`) |
| **feature-flags-go** | p-blackswan/feature-flags-go | Go client for the flag service |
| ~~fixedpoint~~ | p-blackswan/fixedpoint | decimal arithmetic — **dormant** (last push 2026-02); read-mono uses `govalues/decimal`. Don't introduce it in new code |

⚠️ corekit versions **drift hard** across service modules (read-mono alone spans v0.21.3 → v0.41.x).
A corekit fix does not reach a service until that module's `go.mod` is bumped.

## Infrastructure

- **Cloud:** AWS (multi-account, Terragrunt) + Huawei Cloud. Region `eu-central-1`.
- **K8s:** EKS/CCE + ArgoCD (both ArgoCDs run on `plt-prd-utility`, namespaces `argocd` / `argocd-nonprod`).
  Workloads are mostly **Argo Rollouts** (blue-green), not Deployments — `kubectl get deploy` misses them.
  Also in-cluster: KEDA (event-driven autoscaling), Karpenter, external-secrets, vault-secrets-webhook, reloader, clamav.
- **GitOps:** `p-platform/platform-gitops`. ⚠️ `blackswan-gitops` and `blackswan-infrastructure` are **ARCHIVED**.
- **Messaging:** internal Redpanda + AWS MSK + redpanda-connect (see above). Ops UIs: kafka-ui, redpanda-msk-console, kminion (lag exporter → `KafkaConsumerLagHigh`).
- **Data layer** (consistent naming — one store per domain):
  - Postgres RDS: `exc-prod-<domain>.cbqssr0jkkf1.eu-central-1.rds.amazonaws.com` (balance, campaign, commission, custody-integration, financial-history + `-read-1` replica, gopanel, order, pnl, reward, staking, user-state, momentum, …)
  - ElastiCache Redis: `master.exc-prod-<domain>.xeyvt9.euc1.cache.amazonaws.com` (ticker, pnl, orderbook, wapi, ws, wallet, user-state, user-volume, commission, financial-history, custody-integration, anomaly-detection, order-api, order-responder); `momentum` is on **Valkey**
  - **ClickHouse Cloud** over VPC endpoints (`<id>.eu-central-1.vpce.aws.clickhouse.cloud`): 3 separate services — `ticker`, `klines`, `uservolume` (+ `heimdall`)
  - Some services carry a single DSN instead of split vars (e.g. defi uses `DEFI_DB_URL`)
- **Registry:** Harbor `harbor.int.hqplatform.io/exc-<namespace>-<env>/<svc>:<sha>`
- **Secrets:** HashiCorp Vault (`vault:<path>#<key>` placeholders, resolved at PID 1 — see `test-env.md`)
- **Feature flags:** **Flipt v2** (ns `blackswan`, HTTP :8080 / gRPC :9000, `flipt.int.paribu.com`); flag state in `p-blackswan/feature-flags` under `flipt/<env>`. ⚠️ flagd/OpenFeature is **gone** from gitops.
- **Observability:** SigNoz (traces/logs; chart 0.114.1, ns `signoz` on plt-prd-utility, separate `signoz-nonprod`), VictoriaMetrics (`vm-cluster` + `vm-nonprod`), VictoriaLogs (`vl`, k8s events + edge), Pyroscope (`profiling.int.hqplatform.io`), Grafana + grafana-operator, obi (OTel eBPF auto-instrumentation DaemonSet), otel collectors (`otel-cluster-agent-collector.opentelemetry.svc.cluster.local:4317`). Alerting = PrometheusRule CRs in `platform-gitops/monitoring/alerts/pod-<team>/`, evaluated centrally.
- **Access:** everything internal is VPN-only. Telemetry via the **sre-mcp** aggregator (SSO).

## GitHub Orgs

| Org | What's there |
|---|---|
| **p-blackswan** | Core exchange services, shared libs, plugins (110 active / 29 archived repos) |
| **p-platform** | platform-gitops, ansible, service-catalog, sre-mcp |
| **p-onchain** | DeFi/on-chain: p-accounts(-sol), MagicSpend, kernel, 7579-plugins, swig-wallet, legolas, token-list, pario — **and this pack (`architecture-context`)** |
| **p-backoffice** | Paribu One framework + domain repos (`paribu-one`, `-defi`, `-denetim`, `-dys`, `-nexus`, `-chat`, `-saasops`, `-compliance`), helpdesk-core, assistant-core, **kanban** |
| **p-utilities** | Client SDKs & tooling: storybook, chartPro, p-kyc, expo-dtt, passkeys, p-nse, nfc-passport-reader, figsmith, paribu-agent-marketplace |
| **pikachu-exchange** | samaritan (mobile) + legacy/BD repos. ⚠️ gopanel moved to p-blackswan |

## Don't reference these — archived

`blackswan-gitops`, `blackswan-infrastructure`, `auth-service` (the repo — mellon replaced it),
`gateway`, `match-core`, `revoke-relay`, `ws-server`, `notification-templates`, `mkk`,
`web-archived`, `fraud-aware-otp-service`, `order-event-forwarder`, `commission-cost-tracker`,
`finance-dashboard`, `v4-asset-*`, `wallet-match-event-tracker`, all `poc-*`.

## How to Use This

1. Read this INDEX for the big picture
2. Read `services/<name>.md` for the service you're touching — it tells you what it talks to
3. Read `flows/<flow>.md` if you need an end-to-end process
4. Read `conventions.md` for coding standards, ticket/branch rules, and the deploy mechanism
5. Read `test-env.md` to reach the live test cluster (kubectl/auth/VPN) and read its logs, DBs, vault secrets
6. **Clone the repo and read the code** for implementation details — and check
   `.cd/helm/<env>/<app>.yaml`, which is the ground truth for ports, namespaces, brokers, and datastores
