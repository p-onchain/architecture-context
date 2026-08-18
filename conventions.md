# Conventions & Shared Patterns

> Coding standards, naming conventions, and shared patterns across BlackSwan services.
> **Last verified 2026-08-17** against `shared-workflows@main`, `platform-gitops@main`, corekit
> `v0.41.0`, and live `.cd/helm` values.

## Language & Stack

- **Primary language:** Go. Version is **not uniform** and the drift is deliberate during rolling
  upgrades — read-mono alone spans `1.25.3`, `1.25.4` and `1.26.1` across its per-service modules;
  order-api / bff-api / wallet / match are on `1.26`, while corekit / bff-client / notif2 are on
  `1.25.3`. Match the module you're editing; don't bump a `go` directive as a drive-by (corekit's
  version is a floor for every consumer).
- **Exceptions:** onchain + bundler-erc4337 (TypeScript/Bun), heimdall and **estel** (Rust),
  micro-web (Vue), samaritan (React Native), pdf-operations (Python), SoapInstantPay /
  mikro-integrations (C#).
- **Standard toolkit:** `p-blackswan/corekit` — nearly every Go service
- **Observability:** `p-blackswan/observer` — shared OTLP + Pyroscope setup
- **Proto contracts:** `p-blackswan/proto-hub`
- **Helm base:** `p-blackswan/blackswan-helm-base` (chart alias `blackswan-service`)

## Project Structure (Go Services)

```
service-name/
├── .cd/
│   ├── docker/Dockerfile
│   └── helm/
│       ├── Chart.yaml
│       ├── prod/<app>.yaml       # ONE ArgoCD app per values file (see Deployment below)
│       ├── test/<app>.yaml
│       ├── hw-prod/<app>.yaml    # Huawei clusters, when the service runs there
│       └── hw-test/<app>.yaml
├── cmd/main.go
├── internal/
│   ├── config/                   # corekit/envcfg
│   ├── domain/ application/ adapters/
├── go.mod
├── Makefile
├── lefthook.yml
└── CLAUDE.md / AGENTS.md         # per-repo (and in read-mono, per-domain) agent notes
```

`.cd/helm/<env>/<app>.yaml` is the **ground truth** for ports, namespace, Kafka brokers, topics,
consumer groups, datastores, and resource sizing. Read it before believing any doc — including this one.

## Corekit Packages

| Package | Usage |
|---|---|
| `corekit/envcfg` | env-based configuration |
| `corekit/database/{postgres,redis,clickhouse}` | connections + migrations |
| `corekit/messaging/kafka` | consumer (simple/batch processors, DLQ) + producers (confluent-kafka-go/v2) |
| `corekit/grpc`, `grpc/gclient`, `grpc/interceptor` | server, client, standard interceptors |
| `corekit/apperrors`, `xlogger` (zerolog), `logctx` | errors + structured logging |
| `corekit/worker`, `xjobs` | background workers, distributed jobs (pg/redis store) |
| `corekit/retry`, `httptransport`, `masking` | backoff, HTTP setup, log masking |
| `corekit/clock`, `xtime`, `xsync`, `testing` | injectable time, concurrency helpers, test kit |
| `corekit/excluded` | fail-closed, atomically-refreshed cache of **excluded user IDs** (market-makers/bots the read side skips in per-user aggregates) |
| `corekit/notify` | typed client publishing notification requests to notif2's Kafka ingress (`notification/request/v1`) |
| `corekit/userquery` | thin client over user-query `GetBulkUsers`; a data primitive, it does **not** own exclusion policy |

⚠️ **Version drift is the norm.** read-mono service modules pin anything from `v0.21.3` to
`v0.41.x`. A corekit fix is live only in modules whose `go.mod` was bumped — verify per service
before assuming a fix shipped. Local corekit edits also don't affect a service build unless you add
a `replace` (and don't commit it).

## Naming Conventions

### Services
- Lowercase, hyphenated: `order-api`, `wallet-ledger-sink`, `commission-update-worker`
- CQRS pattern: `{domain}-projection`, `{domain}-query`, `{domain}-worker`
- BFF pattern: `bff-api` (web), `bff-client` (mobile), `bff-instant-pay`
- ⚠️ Deployment name ≠ repo name in places: **mellon → `auth-service`**, notif2 → `notify-api`,
  micro-web → `paribu-*`, estel → `kyc-*`, paribu-one-compliance → `frontend`/`service`.

### Kafka Topics
- Match/status/orderbook events are **shared, un-suffixed** — consumers filter by market from the
  payload: `order.events.match`, `order.events.status`, `orderbook.state`, `orderbook.match_price`
- The match-engine **WAL** IS per-market suffixed: `order.requests.{market}`
- Ledger: `ledger-logs`, `external.ledger-logs`
- Domain events: `{domain}-events` (`user-events`, `user-commission-events`, `config-service-events`)
- Newer dotted namespaces: `ticker.daily`, `ticker.last24h`, `pnl.user.asset.updates`,
  `pnl.defi.asset.updates`, `order.events.dex`
- DLQ: `{service}.dlq` (e.g. `balance-projection.dlq`)

### Protobuf
- Package: `proto.{domain}.v1`; newer DEX contract is `proto.dex.order.v2`
- Go package: `github.com/p-blackswan/proto-hub/gen/go/{domain}/v1;{domain}_v1`
- Service names `{Domain}Service`; event names `{entity}_{action}_event`

### Environment Variables
- Uppercase, underscore-separated
- gRPC addresses: `{SERVICE}_ADDR` / `{SERVICE}_SERVICE_URL`; in-cluster gRPC is almost always
  **:50051**. Match engines are templated: `MATCH_SERVICE_ADDR_BASE=match-%s:50051` with
  `%s = {currency}-{payment}`
- DB: `POSTGRES_HOST/PORT/USERNAME/PASSWORD/DATABASE`, `CLICKHOUSE_*`; some services carry one DSN
  instead (`DEFI_DB_URL`)
- Kafka: `KAFKA_BROKERS`, `KAFKA_*_TOPIC`, `KAFKA_*_CONSUMER_GROUP`, `KAFKA_SECURITY_PROTOCOL`,
  `KAFKA_SASL_MECHANISM/USERNAME/PASSWORD`, `SCHEMA_REGISTRY_URL`
- Redis: `REDIS_HOST/PORT/DB/PASSWORD/TLS_ENABLED`
- Observability: `OTEL_*`, `OBSERVER_OTLP_ENDPOINT`, `OBSERVER_PYROSCOPE_SERVER_URL`
- Secrets are Vault placeholders: `vault:<namespace>/data/<service>#<KEY>`

## Communication Patterns

### gRPC (synchronous)
- All inter-service sync calls use gRPC + protobuf from proto-hub
- Standard corekit interceptors (logging, tracing, recovery, auth metadata)
- User ID propagated via gRPC metadata (originally KrakenD's `X-User-Id` header)
- Internal exposure outside the cluster is via Envoy **GRPCRoute** on `internal-gw`:
  `<service>.int.paribu.com` (plaintext, reflection on, VPN-only)

### Kafka (asynchronous)
- **Two brokers.** Internal Redpanda (plaintext `10.240.*:9092`) for the trading hot path; AWS MSK
  (`excprodexternal`, SASL_SSL/SCRAM-SHA-512) for everything else; `redpanda-connect` mirrors
  between them. Full map in `INDEX.md` — pick the right one from the service's helm values, never
  by assumption.
- Consumer groups follow `{service-name}[-{topic-suffix}]`
- Protobuf serialization via the Confluent Schema Registry (see below)
- Because the read side consumes **mirrored** copies, MSK retention bounds replay/rebuild windows

#### corekit processors — commit policies

`corekit/messaging/kafka/consumer/processors/{simple,batch}`; `PROCESSOR_COMMIT_POLICY`:

| Policy | Env value | Semantics |
|---|---|---|
| ManualCommit | `manual` (default) | sync commit after successful handle — at-least-once |
| AutoCommit | `auto` | commit before handle — at-most-once (fire-and-forget) |
| AsyncCommit | `async` | non-blocking offset store after handle; librdkafka commits in background — at-least-once without a per-message broker round-trip. Needs `enable.auto.commit=true` + `enable.auto.offset.store=false` |

#### corekit processors — per-partition head-of-line isolation

Opt-in via `PROCESSOR_PARTITION_ISOLATION=true`. Each partition gets a buffered worker channel;
a slow partition is paused at the broker at high-water and resumed at low-water, so it can't wedge
the shared poll loop.

| Env var | Default |
|---|---|
| `PROCESSOR_PARTITION_ISOLATION` | `false` |
| `PROCESSOR_PARTITION_BUFFER_SIZE` | `1000` |
| `PROCESSOR_PARTITION_ISOLATION_HIGH_WATER` | `0.9` |
| `PROCESSOR_PARTITION_ISOLATION_LOW_WATER` | `0.5` |

Related: `PROCESSOR_HANDLER_TIMEOUT`, `PROCESSOR_MAX_BATCH_SIZE`, `PROCESSOR_MAX_FLUSH_TIMEOUT`,
`PROCESSOR_RETRY_COUNT/BACKOFF`, `PROCESSOR_DLQ_*`, `PROCESSOR_STOP_GRACE_TIMEOUT`.

⚠️ Partition isolation is **incompatible** with DLQ pause-on-failure consumer mode — both on fails
at startup.

**Enabled on:** wapi (`async` + isolation, buffer 256, concurrency 8) and read-mono's open-order-v2 trio.

### Schema Registry
- One self-hosted **Confluent** SR (`cp-schema-registry` 8.0.3, 3 replicas, CFK operator) in ns
  `blackswan`, backed by MSK. In-cluster
  `schema-registry-{0,1,2}.schema-registry.blackswan.svc.cluster.local:8081`; externally
  `https://schema-registry.int.paribu.com`
- `AUTO_REGISTER_SCHEMAS=false` in prod (registered from proto-hub CI), `USE_LATEST_VERSION=true`
- TopicNameStrategy is pinned explicitly by newer corekit (≥ v0.32.0)

## Decimal Handling

- All financial values are **strings** in protobuf (never float/double)
- **read-mono uses `govalues/decimal`** — do NOT introduce `fixedpoint` there. Some newer services
  use `shopspring/decimal`; check the module's `go.mod`.
- `p-blackswan/fixedpoint` is effectively dormant (last push 2026-02). Prefer `govalues/decimal` in new code.

## User IDs

- UUIDv7 throughout, generated server-side; passed as strings in protobuf
- System accounts: user_id `42` (trade commissions), `43` (withdraw commissions)
- Org stance: a bare `user_id` UUID is **not** by itself identifying PII — privacy review focuses
  on content fields, not id-keyed rows

## Feature Flags

**Flipt v2** (ns `blackswan`, HTTP :8080 / gRPC :9000, `flipt.int.paribu.com`), flag state committed
to `p-blackswan/feature-flags` under `flipt/<env>`, Go client `feature-flags-go`, dashboard `ff-dash`.
⚠️ flagd / OpenFeature is gone — don't wire new code to it.

## Git & Ticket Conventions

Tickets live in the internal **Kanban** app (not JIRA). Three layers enforce this; all three must pass.

1. **Branch Name Guard** (`shared-workflows/.github/workflows/branch-guard.yaml`) on every push to
   a non-main branch:
   ```
   <type>/<KEY>-<number>-<kebab-scope>
   type ∈ feat | feat! | fix | chore | refactor | perf | docs | test | build | ci
   KEY  ∈ E1B–E9B | E1PT–E9PT | E1E–E3E | E7D | EXC | EBV | HYD | H1 | H2 | QE | PLA
   e.g. feat/E1B-123-orderbook-iceberg-orders
   ```
2. **PR Title Guard** (`pr-guard.yaml`): `<type>: [<KEY>-123] <summary>` — same key set.
   Commit messages follow the same shape (`feat: [E4B-81] …`); Turkish summaries are common and fine.
3. **Kanban Guard** — an **org ruleset** ("kanban task required") that resolves the keys in the PR
   and requires each to be a **live** task. `p-blackswan/shared-workflows/kanban-guard.yaml` is a
   deliberate pointer to the canonical `p-backoffice/shared-workflows` implementation.

Traps that cost real time:
- The ruleset pins the guard with "require workflows to pass", so GitHub only re-triggers on
  **opened / reopened / synchronize** — a corrected PR title needs a push or reopen to be re-read;
  editing alone leaves the check red.
- branch-guard's key list and the kanban board allowlist can disagree — you need a live card whose
  key is in the **branch** too. `EXC-0`-style placeholders fail kanban-guard.
- **Renaming an open PR's HEAD branch closes the PR unreopenably.** Never rename; open a new one.
- `platform-gitops` uses its own ruleset: branch `<type>/PLA-<ID>[-slug]`, commit
  `<type>[scope][!]: [PLA-<ID>] description`, target `main`, 1 approval.
- **read-mono rejects any push touching `.github/**`** (push ruleset with no bypass actors — the
  *whole push* is rejected). Split those edits out and hand them over.
- `lefthook` pre-commit strips unstaged halves of partially-staged files, so a subset commit fails
  while someone has WIP in the same file.
- Main branch is `main` (never `master`).

## Deployment (how a service actually ships)

ArgoCD ApplicationSet `apps-exchange` in `platform-gitops/app-of-apps/exchange/`:

1. **scmProvider generator** scans the whole **p-blackswan** org for repos containing `.cd/helm`
   (`excludeArchivedRepos: true`)
2. matrixed with clusters labelled `product=exchange, cloud=aws` (`clusterEnv` from the cluster's
   `environment` label, revision from its `branch` label)
3. matrixed with a **git file generator** over `.cd/helm/<clusterEnv>/*.yaml`

⇒ **one ArgoCD Application per values file**, named `<values-file-basename>-<clusterName>`
(a file literally named `values.yaml` yields `<repo>-<clusterName>`).

Consequences worth knowing:
- Adding `.cd/helm/prod/foo.yaml` and merging **creates a new prod app** — no gitops PR needed.
- ArgoCD `destination.namespace` defaults to `exchange`, but the chart's own
  `blackswan-service.namespace` decides where pods land. The ArgoCD-reported namespace is not real.
- `image.tag` diffs are ignored by the ApplicationSet (CI patches tags), and
  `applicationsSync: create-update` + `preserveResourcesOnDeletion: true` mean **apps are never
  pruned** — orphans linger after a repo is archived.
- Rollout strategy is typically **blueGreen** with `autoPromotionEnabled` (Argo Rollouts), so
  `kubectl get deploy` shows nothing; use `kubectl get rollouts`.
- Huawei clusters use the parallel `hw-prod` / `hw-test` value dirs and a separate app-of-apps.

## Observability

- **Tracing:** OpenTelemetry (OTLP) → SigNoz (`otel-cluster-agent-collector.opentelemetry.svc.cluster.local:4317`)
- **Metrics:** OTLP + kube-state/exporters → VictoriaMetrics → Grafana
- **Logs:** structured JSON (zerolog) → SigNoz; **VictoriaLogs** carries k8s events + edge logs
- **Profiling:** Pyroscope (`https://profiling.int.hqplatform.io`)
- **eBPF:** `obi` auto-instrumentation DaemonSet supplements app instrumentation
- Every service sets `OTEL_SERVICE_NAME` to its deployment name; KrakenD reports as **`krand`**
- **Alerts are PrometheusRule CRs in `platform-gitops/monitoring/alerts/`** — team-owned dirs
  `pod-exc-1..9`, `pod-cus-*`, `pod-data-1`, `pod-hyd-*`, `pod-pass-*`, `pod-trd-*`, plus `dba`,
  `sre`, `platform-exchange`, `platform-custody`, `generic`, `no-op`, `_recording`.
  ⚠️ Grafana alert YAML committed inside a service repo pages nobody. Labels use
  **priority P1–P4**, not `severity`.
- Access is via the **sre-mcp** aggregator (SSO, VPN-only) — `sre_investigate` first.

## CI/CD

- GitHub Actions with reusable workflows from **p-blackswan/shared-workflows**:
  `branch-guard`, `pr-guard`, `kanban-guard`, `golangci-lint`, `gosec`, `test`, `trivy`, `gitleaks`,
  `hadolint`, `dive`, `kube-linter`, `kube-score`, `yaml-lint`, `release`, `release-x64`,
  `create-release-tag`, `validate-workflows`, `claude`, `stale`
- Docker build → Harbor (`harbor.int.hqplatform.io/exc-<namespace>-<env>/<svc>`) → ArgoCD sync
- Linting locally: `golangci-lint` + `lefthook` hooks. Keep the local golangci version aligned with
  CI's — version drift produces unsatisfiable findings.
- ⚠️ **A green CI run can prove very little.** In read-mono the `golangci-lint` step is a permanent
  no-op (it consumes a bash array built in another step) and there is **no `go test` job** — green
  doesn't even prove compilation. Build and test locally; when comparing scanner output, diff
  counts-by-family against another PR rather than treating pre-existing failures as yours.
- Some repos (e.g. notif2) have **no required status checks** on `main`, and a skipped job reports
  as SKIPPED rather than failed.

## Testing

- Unit: `go test ./...` (per module in read-mono — `go.work` is local-dev convenience only)
- Integration: Docker Compose (`compose.yml` per repo)
- Load: k6 (`bswn-k6-suite`); E2E: `exchange-automation-tests` (TS, order→wallet→match)
- Test data: `market-simulator` generates synthetic activity on the internal test exchange
- ⚠️ A test written alongside its own fix inherits the fix's blind spot — mutation-killing proves
  the test detects a change, not that the pinned behaviour is correct.
