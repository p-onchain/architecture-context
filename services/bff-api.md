# bff-api

> Backend-for-Frontend serving the web client. Gateway proxy that aggregates gRPC microservices into HTTP/JSON responses.

## Quick Facts

- **Repo:** p-blackswan/bff-api
- **Language:** Go
- **Port:** HTTP :3001
- **DB:** None (stateless proxy)
- **Sits behind:** KrakenD Gateway

## What It Does

1. Receives HTTP requests from KrakenD (which handles JWT validation)
2. Fans out to downstream gRPC microservices
3. Aggregates responses into client-shaped JSON
4. Returns to KrakenD → client

## Downstream gRPC Services

| Category | Services |
|----------|----------|
| **Order** | order-api (HTTP :9000), conditional-order (:50052), open-order-query |
| **User** | user-service, user-query, user-state-query |
| **Balance** | balance-query, wallet |
| **Transaction** | transaction, transaction-query |
| **Market Data** | ticker-query, klines-query, global-price-tracker, orderbook-query |
| **Finance** | commission-api, financial-history-query, pnl-query |
| **Features** | alarm-query, favourite-query, feedback-query, input-validator |
| **Staking** | staking-command, staking-query |
| **Notification** | notification-api |
| **Config** | config-service (HTTP REST) |

## Kafka

- Does NOT consume or produce Kafka messages directly

## Architecture Notes

- Stateless — purely a proxy/aggregation layer
- Uses feature flags (flagd/OpenFeature) for progressive rollouts
- Redis used for MFA session state
- Same env var naming convention as bff-client for downstream services
- Config service accessed via HTTP REST (not gRPC)

## Related

- **bff-client** — same pattern but for the mobile app (Samaritan). Slightly different response shapes optimized for mobile.
- **krakend-gateway** — sits in front, handles JWT validation, rate limiting, API-key auth
