# WAPI (Trading WebSocket API)

> Low-latency WebSocket service for API-key trading clients.

- **Repo:** p-blackswan/wapi
- **Lang:** Go
- **Auth:** API-key on `/v1/user` (KrakenD HMAC), anonymous on `/v1/stream`
- **Data source:** Internal Redpanda directly

## Endpoints

- `/v1/stream` — public market data (orderbook, ticker, trades, klines)
- `/v1/user` — private user data (balances, order fills, status changes)

## Key Details

- Independent from ws-hub — completely separate implementation
- Hexagonal architecture (`internal/{domain,application,adapters,config}/`)
- Flat JSON wire format with single-letter abbreviations
- Consumes directly from Redpanda (not via MSK bridge)
