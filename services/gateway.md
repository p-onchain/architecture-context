# Edge & API Gateway

> Two layers, not one: **Envoy Gateway** (Kubernetes Gateway API) terminates and routes; **KrakenD**
> does JWT/API-key auth, rate limiting, request logging and endpoint composition.

**Last verified 2026-08-17.**

## Layer 1 — Cloudflare

TLS termination, WAF, DDoS shielding. ⚠️ Edge logs are **not** exposed via sre-mcp — if a symptom
could be edge-side, say so rather than guessing from in-cluster data.

## Layer 2 — Envoy Gateway (Gateway API)

- Config lives in **`p-platform/platform-gitops`**: `applications/gateway-resources/values/<cluster>.yaml`
  (Gateways, listeners, NLB annotations, source-range ACLs) and
  `applications/http-routes/values/<cluster>.yaml` (HTTPRoute / GRPCRoute / SecurityPolicy /
  BackendTrafficPolicy). Not in any service repo.
- Namespace `gateway-api-system`; NLB in front, TLS via ACM certs. Three Gateways:

| Gateway | Purpose |
|---|---|
| `external-gw` | public traffic (listeners `https`, `https-paribu-apex`, `https-paribu-services`) |
| `internal-gw` | in-VPC exposure of ~58 services as `<service>.int.paribu.com`; gRPC on `sectionName: grpc` via **GRPCRoute** |
| `internal-ops-gw` | ops UIs — grafana, vmalert, vault, kafka-ui, redpanda-msk-console, schema-registry, notification-ops |

**Public routing:**

```
web.paribu.com, api.paribu.com, app.paribu.com, ext.paribu.com,
mobile.paribu.services, web-v6/api-v6/app-v6.paribu.com   → krakend-gateway:8080
paribu.com, www.paribu.com                                → paribu-orchestrator:8080 (micro-web)
   ├─ /blog → support-external      (external EndpointSlice)
   └─ /hub  → hub-external          (external EndpointSlice)
go-v6.ceteris.io                                          → gopanel-frontend / gopanel-backend
schema-registry.paribu.com                                → schema-registry (basic-auth SecurityPolicy)
+ helpdesk WS, dash-frontend, webhelp, kep external routes
```

**Internal routing** — a GRPCRoute per service, in the service's own namespace, e.g.
`ticker-query.int.paribu.com` (ns `saul`), `pnl.int.paribu.com`, `defi-query.int.paribu.com`,
`user-state-query.int.paribu.com`, `wallet.int.paribu.com`, `auth.int.paribu.com`,
`notify-api.int.paribu.com`, `portfolio-query.int.paribu.com`, `flipt.int.paribu.com`.
Plaintext with gRPC reflection enabled → `grpcurl` works over VPN for one-off prod lookups.
Keep PII out of anything you print.

## Layer 3 — KrakenD

- **Repo:** p-blackswan/krakend-gateway · **Tech:** KrakenD (Go) + custom plugins · **Port:** :8080
- ⚠️ **Reports to SigNoz as `service.name=krand`**, not `krakend`.
- Endpoint definitions live in `endpoints/<group>/` — ~600 endpoints across 21 groups:
  addresses, alarm, api, auth, bff_client, customer, financial-history, health, initials, kyc,
  market, notification, openid, orders, partner, staking, support, tickets, transactions, user,
  wapi, websocket. Config is assembled via `flexible_config.json` + `templates/` + `generators/`.
- Upstreams: `bff-api:3000` (web), `bff-client:8080` (mobile), `bff-instant-pay`, wapi WS, plus
  direct service routes.
- Validates JWTs issued by **mellon** (deployed as `auth-service`); `krakend-revoke-server` handles
  revocation propagation. ⚠️ `revoke-relay` is **archived** — cross-cluster invalidation no longer
  goes through it.

### Custom plugins
- **krakend-apikey-plugin** — HMAC API-key auth, tiered/weight-based rate limiting, IP whitelisting
- **krakend-requestlog-plugin** — request/response logging with data masking (feeds `audit-log-api`,
  which stores KrakenD request logs in TimescaleDB)

### Behaviors worth knowing
- **Rate limiting exists only here.** In-cluster callers that hit `order-api:3000` or any
  `*.int.paribu.com` route bypass it entirely.
- **Dual-auth routes:** some endpoints require both a user JWT and an `X-Internal-Token`
  (e.g. `/conversion/cost-basis`) — KrakenD forwards both to the upstream BFF, which validates both.
- `/v1/banners` → bff-client (hardcoded home banners); `/v1/onchain/health` → bff-client (public,
  IP rate-limited 60 req/min); `/v1/onchain/.../challenge` forwards `X-Forwarded-For` for audit.
- **Helpdesk attachments** (`/api/v1/customer/messages/:id/attachments`): 25 MiB cap, 60s timeout.
  `Transfer-Encoding: chunked` is rejected (`body_size_allow_chunked: false`) because the CEL
  body-size guard needs `Content-Length`. bff-client's `http.MaxBytesReader` is the authoritative cap.
  ClamAV `MaxFileSize` is 30 MiB — raising past 25 MiB needs a coordinated ClamAV configmap change.
- Agentic coding configs live in the repo root (`.claude/`, `.cursor/`, `.codex/`, `AGENTS.md`).
