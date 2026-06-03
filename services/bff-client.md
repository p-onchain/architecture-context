# bff-client

> Backend-for-Frontend for the mobile app (Samaritan). Same pattern as bff-api but with response shapes optimized for mobile.

- **Repo:** p-blackswan/bff-client
- **Lang:** Go
- **Port:** HTTP :8080
- **Stateless proxy** — no DB

## Talks To

Same downstream services as bff-api (see `bff-api.md`), with mobile-specific response mapping.

## Key Details

- Sits behind KrakenD
- No Kafka
- Uses corekit + proto-hub
- Feature flags via flagd/OpenFeature
