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

## Consumer Configuration (as of 2026-06)

| Env var | prod | test | Notes |
|---------|------|------|-------|
| `WAPI_CONSUMER_CONCURRENCY` | **8** (was 3) | **4** (was 3) | ADR-008 operational cap is 8; raised after 2026-06-08 head-of-line incident. At concurrency=3 each member owned ~21 partitions of the 64-partition hot topics — one stalled partition starved its siblings. |
| `WAPI_PROCESSOR_PARTITION_BUFFER_SIZE` | **256** (was 64) | **256** (was 64) | Drain cushion; old 64 cap was based on now-obsolete `orderbook.state` consumption (dropped, ADR-011) |
| `PROCESSOR_COMMIT_POLICY` | `async` | `async` | Non-blocking offset store — at-least-once without per-message broker round-trip (2026-06) |
| `PROCESSOR_PARTITION_ISOLATION` | `true` | `true` | Per-partition head-of-line isolation — slow partitions paused at broker (2026-06) |

> **ADR-008 note:** `WAPI_CONSUMER_CONCURRENCY=8` is the operational cap for prod. test env is limited to 4 due to its 256Mi/0.5-core budget.
