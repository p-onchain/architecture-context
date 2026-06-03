# KrakenD Gateway

> API gateway. Handles JWT validation, rate limiting, API-key authentication, request logging, and routing to BFF services.

## Quick Facts

- **Repo:** p-blackswan/krakend-gateway
- **Technology:** KrakenD (Go-based API gateway) with custom plugins
- **Sits between:** Clients ↔ BFF services

## What It Does

1. **JWT Validation** — validates tokens issued by mellon on every request
2. **API-Key Auth** — HMAC-based API key authentication for trading API clients (via krakend-apikey-plugin)
3. **Rate Limiting** — tiered rate limiting with IP whitelisting and weight-based quotas
4. **Request Logging** — logs requests/responses with sensitive data masking (via krakend-requestlog-plugin)
5. **Routing** — routes to bff-api (web), bff-client (mobile), or direct services

## Custom Plugins

- **krakend-apikey-plugin** (p-blackswan) — API key HMAC authentication with tiered rate limiting, IP whitelisting, weight-based quotas
- **krakend-requestlog-plugin** (p-blackswan) — request/response logging with sensitive data masking, S3 sink

## Routing

```
/api/v1/web/*     → bff-api:3001      (web client)
/api/v1/mobile/*  → bff-client:8080   (mobile client)
/v1/stream        → wapi              (public WS, anonymous)
/v1/user          → wapi              (private WS, API-key auth)
```

## Related Services

- **krakend-revoke-server** — propagates token revocation to KrakenD instances
- **revoke-relay** — cross-cluster session invalidation
