# bff-api

> Backend-for-Frontend for the web client. Aggregates gRPC microservices into HTTP/JSON.

- **Repo:** p-blackswan/bff-api
- **Lang:** Go
- **Port:** HTTP :3000
- **Stateless proxy** — no DB

## Talks To

Sits behind KrakenD, fans out to downstream gRPC services:
- **Trading:** order-api, conditional-order, open-order-query
- **User:** user-service, user-query, user-state-query
- **Balance:** balance-query, wallet
- **Transaction:** transaction, transaction-query
- **Market Data:** ticker-query, klines-query, global-price-tracker, orderbook-query
- **Finance:** commission-api, financial-history-query, pnl-query
- **Features:** alarm-query, favourite-query, feedback-query, input-validator
- **Staking:** staking-command, staking-query
- **Notification:** notification-api
- **Config:** config-service (HTTP REST)
- **Support:** tickbu (HTTP REST — support ticket system)

## Key Details

- No Kafka — purely a proxy/aggregation layer
- Uses feature flags (flagd/OpenFeature)
- Redis for MFA session state
- bff-client is the same pattern but for mobile (Samaritan)
