# KrakenD Gateway

> API gateway. JWT validation, rate limiting, API-key auth, request logging, routing.

- **Repo:** p-blackswan/krakend-gateway
- **Tech:** KrakenD (Go) + custom plugins

## Routing

```
/api/v1/web/*     → bff-api:3001
/api/v1/mobile/*  → bff-client:8080
/v1/stream        → wapi (public WS, anonymous)
/v1/user          → wapi (private WS, API-key auth)
```

## Custom Plugins

- **krakend-apikey-plugin** (p-blackswan) — HMAC API-key auth, tiered rate limiting, IP whitelisting
- **krakend-requestlog-plugin** (p-blackswan) — request/response logging with data masking

## Key Details

- Validates JWTs issued by mellon
- krakend-revoke-server handles token revocation propagation
- revoke-relay handles cross-cluster session invalidation
