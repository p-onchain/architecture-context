# bff-client

> Backend-for-Frontend for the mobile app (Samaritan). Aggregates ~25 gRPC microservices into mobile-optimized JSON responses.

## Quick Facts

- **Repo:** p-blackswan/bff-client
- **Language:** Go
- **Port:** HTTP :8080
- **DB:** None (stateless proxy)
- **Sits behind:** KrakenD Gateway

## What It Does

Same role as bff-api but optimized for the mobile client:
1. Receives HTTP requests from KrakenD
2. Fans out to gRPC microservices
3. Returns mobile-shaped JSON responses

## All Downstream Services

**Order:** order-api, conditional-order, open-order-query
**User:** user-service, user-query, user-state-query
**Balance & Transaction:** balance-query, transaction, transaction-query, commission-api, uservolume-query
**Market Data:** ticker-query, klines-query, global-price-tracker, orderbook-query
**History:** financial-history-query, pnl-query, pnl-agg-query
**Alarm & Favorites:** alarm-query, favourite-query
**Staking:** staking-command, staking-query
**Notification & Support:** notification-api, support, feedback-query
**Validation:** input-validator (gRPC)
**External Data:** heimdall (sentiment, market overview — Rust service)
**Config:** config-service (HTTP REST)
**Helpdesk:** helpdesk-backend (internal s2s auth)
**Stock Trading:** xpr-stock services (US + BIST asset, market-data, watchlist, alert APIs)

## Mobile-Specific Features

- Curated market lists (S3-backed, cached in-memory)
- "Bought together" related assets (S3-backed)
- Heimdall integration for Fear & Greed Index, global metrics
- Stock trading surfaces (BIST/US) with per-market kill switches

## Architecture Notes

- Same env var convention as bff-api (`*_API_URL` for gRPC, `*_API_URL` with `http://` for HTTP)
- ORG_SERVICE_TOKEN for helpdesk s2s auth (Vault-sourced in prod)
- Uses corekit for standard patterns
