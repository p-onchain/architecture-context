# Test environment — phoenix-test (kubectl) & nonprod telemetry

**Last verified 2026-08-07** (access recipes) / **2026-08-17** (deploy mapping below).

> **How test relates to prod.** Test mirrors prod's namespace layout (`blackswan`, `saul`, `shelby`,
> `onchain`, `gopanel`, `corleone`, `momentum`) — see `INDEX.md` for the prod cluster/namespace map and
> the two-Kafka topology. Which apps exist per env is decided by the values files in each repo:
> `.cd/helm/test/*.yaml` → `exc-test-alpha` (this cluster), `.cd/helm/hw-test/*.yaml` → `exc-test-hw`
> (Huawei). One ArgoCD app per file; adding a file creates an app on merge.
>
> Test is **not** a subset of prod. read-mono, for example, has 49 test apps vs 43 prod, including
> `open-order-v2-{projection,query}` and `pass-projection` that don't exist in prod, plus
> `transaction-query`, `user-query` and `bank-integration-query` which in **prod** run only on Huawei.
> The read-mono `notify` domain (`notify-projection`/`notify-query`) is Huawei-only in both envs.

Two ways to observe the test environment:
- **Telemetry** — the **sre-mcp** aggregator (plugin `sre-mcp@paribu-agent-marketplace`,
  SSO-authed): `signoz_nonprod_*` (traces/logs) + `vm_nonprod_*` (PromQL) expose the
  telemetry of the `phoenix-test` cluster below (the SigNoz service list matches these
  workloads 1:1); `vlogs_*` carries **k8s events** (filter `log.type="k8s-event"` +
  `k8s.cluster.name`), `argocd_*` deploy state, `alertmanager_*` active alerts.
  (Legacy fallback: the direct `signoz-nonprod` / `victoriametrics-nonprod` MCP servers —
  deprecated, prefer sre-mcp.)
- **Live pods** — `kubectl` into EKS `phoenix-test` (this doc).

For what-talks-to-what, see `INDEX.md` / `services/<name>.md`. This doc is the *access*
layer: how to reach the cluster, its layout, and how to read its secrets and DBs.

## Prerequisite — VPN

The MCP endpoints (sre-mcp + legacy, all `*.int.hqplatform.io`) and the `phoenix-test` EKS API are internal — they
resolve only over the user's VPN. VPN-off symptom: DNS `no such host` / connection timeouts
(distinct from an expired SSO token). I can't bring the VPN up — surface it to the user.
(The `aws sso login` browser flow is public AWS and works without VPN; everything downstream
needs it.)

## Cluster & auth

- Cluster `phoenix-test` (EKS, `eu-central-1`, account `839822588106`).
- **Use context `exc-test-alpha`** — its kubeconfig exec pins `AWS_PROFILE=phoenix-test`,
  so plain `kubectl` works with no prefix. (The bare `phoenix-test` alias context has
  `env: null` → there you must prefix every call with `AWS_PROFILE=phoenix-test`.)
- AWS profile `phoenix-test` → SSO session `phoenix` (`https://d-99672c4332.awsapps.com/start`),
  role `Phoenix-Test-ReadOnly`. Other contexts exist: `phoenix-dev`, `argonaut`.
- ⚠️ **`Phoenix-Test-ReadOnly` is a misnomer.** It grants `kubectl exec` and
  `create pods` (incl. ephemeral debug containers), and the app DB users are read/write.
  Treat the env as mutable — be careful what you run.

### Auth
`aws sso login --sso-session phoenix` (≡ `--profile phoenix-test`) opens a browser, so the
user runs it — prompt them when kubectl 255s. CLI v2 auto-refreshes, so one login lasts the
whole IdC session (~8h); mid-session re-auth is rarely needed.

### Failure signatures
| Symptom | Cause | What I do |
|---|---|---|
| `no such host` / timeouts to `*.int` or the EKS endpoint | VPN off | tell the user — can't fix it myself |
| `aws … exit code 255`, "Token has expired and refresh failed" | SSO session ended | prompt the user to run `aws sso login --sso-session phoenix` |
| `NoCredentials` right after login | stale `AWS_ACCESS_KEY_ID`/`SECRET`/`SESSION_TOKEN` override the profile | `unset` those three myself |

## App namespaces → domain (+ datastore)

| Namespace | Domain | Datastore |
|---|---|---|
| `blackswan` | Core exchange — balance, match-forge, conditional-order, order/wallet, bff, krakend gateway, web | Postgres RDS `blackswan-test` (dbs: balance, match_forge, wallet, order_api, conditional_order) |
| `saul` | CQRS **read side** — `*-projection`/`*-query` (commission, cost-basis, pnl, ticker, orderbook, klines, financial-history, uservolume, notification, **defi**) | **ClickHouse Cloud** (db `ticker`, …) + Postgres for some (defi → Postgres RDS `saul-test` db `defi`, owned by `pnl_admin`) |
| `shelby` | Accounts/auth/financial — auth-service, user-service, bank-integration, crm, transaction, campaign, custody, sanctions | Postgres RDS `shelby-test` |
| `onchain` | DeFi — `defi-*` (api, magic-spend, bundler-base/bsc, hyperliquid/polymarket-stream, scheduler-critical/periodic) | Postgres RDS `onchain-test` (db `onchain_db`) |
| `gopanel` | Admin panel + analytics — gopanel-backend, dash-*, anomaly-detection-* | — |
| `corleone` | Alarms, staking, ws/wapi, mkk, partner-integration | — |
| `momentum` | `exc-momentum` product — heimdall, lssr | ClickHouse + Postgres |

**Two `onchain_db` schema traps** (verified 2026-07-29, cost real time):
- `balance_positions` is keyed by **`account_id`** (bigint), *not* `user_id` — join
  `accounts` (`accounts.id` → `bp.account_id`, `accounts.user_id` is the UUID).
- **Token decimals live in `token_contracts.decimals`**, keyed by
  (`contract_address`, `chain_reference`) — `tokens` holds only id/symbol/name/logo_url
  and has **no address column**. `balance_positions.balance` is raw base units, so you
  need those decimals to compare against anything human-readable.

Infra namespaces: `monitoring`, `vm` (VictoriaMetrics), `opentelemetry` (+ `-operator`),
`obi` (OTel eBPF auto-instrumentation, 131-pod DaemonSet), `argo-rollouts`, `vault`,
`gateway-api-system`.

## Workloads & logs

- Most services are **Argo Rollouts**, not Deployments — `onchain`, `saul`, `gopanel`
  have **no** Deployments. Use `kubectl get rollouts -n <ns>` for real status;
  `kubectl get deploy` silently misses them.
- Stable selector: **`app.kubernetes.io/name=<service>`** (Helm-managed). Images are
  `harbor.int.hqplatform.io/exc-<namespace>-test/<svc>:<sha>`.
- Logs (the `-l` form defaults to `--tail=10` *per pod* and silently truncates — always
  pass `--tail=-1`):
  ```bash
  kubectl -n <ns> logs -l app.kubernetes.io/name=<svc> --since=15m --tail=-1 --prefix
  kubectl -n <ns> logs -l app.kubernetes.io/name=<svc> --previous --tail=500   # after a crash
  ```
- For full-text search / aggregated traces over longer windows, prefer sre-mcp's
  `signoz_nonprod_*` tools over kubectl tailing. For restart/eviction/scheduling *reasons*
  (BackOff, Killing, FailedScheduling…), `vlogs_*` k8s events beat both.

## Secrets / vault (platform-wide pattern)

Every service pod has: `readOnlyRootFilesystem: true`, a `copy-vault-env` init container,
`/vault/` mounted, and `vault:<path>#<key>` placeholder env vars.

- **Secrets resolve only at PID 1.** After `kubectl exec`, `env` shows the unresolved
  `vault:…` placeholders. The **real values are in `/proc/1/environ`** (NUL-separated):
  ```bash
  tr '\0' '\n' < /proc/1/environ | sed -n 's/^DATABASE_PASSWORD=//p'   # one value
  ```
- ⚠️ **The `tr '\0' '\n'` recipe above silently corrupts MULTI-LINE values.** It cannot tell an
  env-entry separator from a newline inside a value, so `sed` stops at the first line. Verified
  2026-08-07 on `saul/notify-api`'s `FCM_CREDENTIALS_JSON` (a service-account JSON stored
  pretty-printed): the extraction yielded exactly `{` — 2 bytes — and the failure surfaced only as
  a downstream `unexpected end of JSON input`. Safe for scalars (hosts, passwords, project ids);
  for anything that may contain newlines, split on NUL only. No busybox tool does this well
  (no `awk`, no `sed -z`, no `python3` in these images), so parse it in the program that needs it:
  `strings.Split(string(os.ReadFile("/proc/1/environ")), "\x00")` — see
  `paribu/plans/tools/la-probe` (`-creds-pid1`).
- **Rootfs is read-only.** Stage any scratch files in `/dev/shm` (or `/vault`), never `/tmp`
  (`/dev/shm` is 64 MB and **not** `noexec`, so staged binaries do run there).
- ⚠️ **`kubectl cp` of a large binary silently truncates** (~28 MB failed repeatedly with
  `write: broken pipe`, leaving a partial file that segfaults in-pod). Retry until the size
  matches — but **size is not a reliable version check**: two different builds of the same Go
  program came out byte-identical. Verify functionally instead, e.g. `<binary> -h | grep -c -- -newflag`.

## Reading a DB via SQL (method depends on runtime)

Connection params live in each pod's `/proc/1/environ`
(`*_HOST` / `*_PORT` / `*_USERNAME` / `*_PASSWORD`, e.g. `DATABASE_*`, `POSTGRES_*`,
`CLICKHOUSE_*`). Keep secrets in-pod — read them inside the exec'd command, never echo them.

⚠️ **Not every service splits them.** Some carry a single ready-made DSN instead, and
grepping for `POSTGRES|DATABASE` then finds nothing useful. Verified: `saul/defi-query`
exposes **`DEFI_DB_URL`** (a full `postgres://…` URL) and nothing else — pass it straight
to `psql "$D"`. So always list the names first, and filter out the Kubernetes service
links or you drown in `*_SERVICE_HOST` / `*_PORT_*_TCP`:
```bash
kubectl -n <ns> exec "$POD" -c <ctr> -- sh -c \
  'tr "\0" "\n" </proc/1/environ | sed -n "s/=.*//p" | grep -vE "_PORT|_SERVICE_|_TCP" | sort -u'
```

💡 **Quoting SQL string literals inside `sh -c '…'`.** SQL needs single quotes for
literals, but the whole command is already single-quoted, so they collide (and SQL
double quotes are *identifiers*, not strings). Use Postgres **dollar-quoting** and the
nesting problem disappears:
```bash
psql "$D" -tAc "select … where table_schema=\$\$defi\$\$ and table_name=\$\$user_dex_holdings\$\$"
```

**1. Bun services (e.g. `onchain/defi-*`)** — Bun has a built-in SQL client:
```bash
POD=$(kubectl -n onchain get pods -l app.kubernetes.io/name=defi-api -o jsonpath='{.items[0].metadata.name}')
kubectl -n onchain exec -i "$POD" -- sh -c 'cat > /dev/shm/q.ts && cd /app && bun run /dev/shm/q.ts; rm -f /dev/shm/q.ts' <<'TS'
const env = Object.fromEntries(new TextDecoder().decode(await Bun.file("/proc/1/environ").bytes())
  .split("\0").filter(Boolean).map(s => { const i = s.indexOf("="); return [s.slice(0,i), s.slice(i+1)]; }));
const { SQL } = await import("bun");
const sql = new SQL(`postgres://${encodeURIComponent(env.DATABASE_USER)}:${encodeURIComponent(env.DATABASE_PASSWORD)}@${env.DATABASE_HOST}:${env.DATABASE_PORT}/${env.DATABASE_NAME}?sslmode=require`);
console.log(await sql`select count(*) from information_schema.tables where table_schema='public'`);
await sql.end();
TS
```

Verified table homes (saves a hunt): **notif2 = `saul/notify-api`** (the rollout named
`notification-*` is the OLD v1 service), Postgres db **`notify`**, device registry table
**`devices`** (`user_id`, `token` = the FCM registration token, `platform`, `token_status`,
`notification_permission`, `deleted_at`, …). ⚠️ One user can hold **several `active` ios rows**
(a stale one plus a fresh one) — `order by updated_at desc limit 1` is the only safe pick, and
`last_seen_at` can be months stale while the row still says `active`.

**2. Go services on Postgres (e.g. `blackswan`, `shelby`)** — no client in the image
(only `wget`). Attach an **ephemeral debug container** that shares the pod's process
namespace (so it can read `/proc/1/environ`) and brings its own `psql`:
```bash
POD=$(kubectl -n blackswan get pods -l app.kubernetes.io/name=balance-query -o jsonpath='{.items[0].metadata.name}')
kubectl debug -n blackswan "$POD" --target=balance-query --image=postgres:17-alpine --container=dbg -i -- sh -c '
  H=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^POSTGRES_HOST=//p")
  U=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^POSTGRES_USERNAME=//p")
  PGPASSWORD=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^POSTGRES_PASSWORD=//p"); export PGPASSWORD PGSSLMODE=require
  psql -h "$H" -U "$U" -d postgres -tAc "select datname from pg_database where datistemplate=false order by 1"
'
# output may not stream over -i; read it back with:
kubectl -n blackswan logs "$POD" -c dbg
```
(Public images pull fine. Ephemeral containers can't be removed — they clear on pod restart.)

**3. ClickHouse services (e.g. `saul`, `momentum`)** — use the pod's own `wget` against the
HTTPS interface on **port 8443** (the env's `:9440` is the native TLS port); creds as headers:
```bash
POD=$(kubectl -n saul get pods -l app.kubernetes.io/name=ticker-query -o jsonpath='{.items[0].metadata.name}')
kubectl -n saul exec "$POD" -- sh -c '
  H=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^CLICKHOUSE_HOST=//p")
  U=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^CLICKHOUSE_USERNAME=//p")
  PW=$(tr "\0" "\n" </proc/1/environ | sed -n "s/^CLICKHOUSE_PASSWORD=//p")
  wget -q -O- --header="X-ClickHouse-User: $U" --header="X-ClickHouse-Key: $PW" "https://$H:8443/?query=SHOW%20DATABASES"
'
```

## When to reach for what

All telemetry via **sre-mcp** (`mcp__plugin_sre-mcp_sre__*`); service-scoped investigations
start with `sre_investigate` (catalog + alerts + golden signals in one call).

- **Traces / aggregated logs (full-text, long windows)** → `signoz_nonprod_*`.
- **Metrics (PromQL)** → `vm_nonprod_*`.
- **K8s events (restart/eviction/scheduling reasons)** → `vlogs_*` with
  `log.type="k8s-event"` + `k8s.cluster.name`.
- **Deploy state/history ("what shipped when")** → `argocd_*`; Grafana deploy annotations
  via `grafana_query_annotations`.
- **Active alerts / silences** → `alertmanager_*`.
- **Live pod state, ad-hoc logs, DB/secret inspection, running one-off scripts** → kubectl
  (this doc). CloudWatch is **not** wired for app logs.
- ⚠️ sre-mcp's own cluster catalog (`sre_list_clusters`, generated 2026-06-24) marks
  `phoenix-test` "unreachable / may be defunct" and lists `exc-test-alpha` as a separate
  nonprod cluster — its ArgoCD registration is stale; kubectl access via context
  `exc-test-alpha` → `phoenix-test` works fine. Verify `k8s.cluster.name` label values
  against actual vlogs/vm data rather than the catalog.
