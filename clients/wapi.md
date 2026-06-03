# WAPI (Trading WebSocket API)

> Low-latency WebSocket service for API-key trading clients. Streams platform events directly from internal Redpanda.

## Quick Facts

- **Repo:** p-blackswan/wapi
- **Language:** Go
- **Architecture:** Hexagonal (`cmd/`, `internal/{domain,application,adapters,config}/`)
- **Auth:** API-key on `/v1/user` (KrakenD HMAC validates), anonymous on `/v1/stream`
- **Data source:** Internal Redpanda directly (not via MSK bridge)

## Endpoints

- `/v1/stream` — public market data (anonymous, no auth)
  - Orderbook snapshots, ticker updates, recent trades, klines
- `/v1/user` — private user data (API-key auth required)
  - Balance updates, order fills, order status changes

## Wire Format

Flat JSON envelope with single-letter abbreviations:
```json
{
  "e": "trade",          // event type
  "E": 1717420800000,    // timestamp (ms)
  "s": "btc-try",        // symbol (optional)
  "r": { ... }           // result payload
}
```
- Numeric timestamps in milliseconds
- Decimals as strings (financial precision)

## Stack

- Kafka: corekit/messaging/kafka (confluent-kafka-go/v2)
- WebSocket: gorilla/websocket
- Proto: proto-hub generated Go structs + corekit Schema Registry serde
- Observability: observer (OTLP + Pyroscope)

## Key Differences from ws-hub

- **wapi:** API-key auth, direct Redpanda consumption, trading-optimized
- **ws-hub:** JWT auth, consumer via MSK bridge, general-purpose for web/mobile
- wapi is NOT a fork of ws-hub — completely independent implementation
