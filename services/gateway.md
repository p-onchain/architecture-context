# KrakenD Gateway

> API gateway. JWT validation, rate limiting, API-key auth, request logging, routing.

- **Repo:** p-blackswan/krakend-gateway
- **Tech:** KrakenD (Go) + custom plugins

## Routing

```
/api/v1/web/*                                → bff-api:3000
/api/v1/mobile/*                             → bff-client:8080
/v1/stream                                   → wapi (public WS, anonymous)
/v1/user                                     → wapi (private WS, API-key auth)
/conversion/cost-basis                       → bff-api (JWT + X-Internal-Token; low global rate limit) [EXCH-6876, 2026-06]
/v1/banners                                  → bff-client (home banners) [EXCH-5185, 2026-06]
/v1/onchain/health                           → bff-client (public; IP rate-limit 60 req/min) [2026-06]
/v1/onchain/.../challenge (POST)             → bff-client (X-Forwarded-For forwarded) [2026-06]
/api/v1/customer/messages/:id/attachments    → bff-client (25 MiB cap, 60s timeout; chunked rejected) [EXCH-6915, 2026-06]
```

## Custom Plugins

- **krakend-apikey-plugin** (p-blackswan) — HMAC API-key auth, tiered rate limiting, IP whitelisting
- **krakend-requestlog-plugin** (p-blackswan) — request/response logging with data masking

## Key Details

- Validates JWTs issued by mellon
- krakend-revoke-server handles token revocation propagation
- revoke-relay handles cross-cluster session invalidation
- **Dual-auth routes:** some endpoints require **both** a user JWT and an `X-Internal-Token` (e.g. `/conversion/cost-basis`) — KrakenD forwards both headers to the upstream BFF
- **Transfer-Encoding: chunked** is rejected at the gateway for the helpdesk attachment endpoint (`body_size_allow_chunked: false`) — CEL body-size guard only works when `Content-Length` is present (EXCH-6915)
- **ClamAV constraint:** helpdesk attachment cap deliberately kept below 30 MiB (ClamAV `MaxFileSize`); raising past 30 MiB requires coordinated ClamAV configmap + pod resource change
- Agentic coding configs present in repo root (`.claude/`, `.cursorrules`, `codex.md`) as of 2026-06
