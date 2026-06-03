# order-api

> Order entry gateway. Accepts orders, validates, reserves funds via wallet, routes to matching engine.

- **Repo:** p-blackswan/order-api
- **Lang:** Go
- **Ports:** HTTP :9000, gRPC :60060

## Talks To

- **wallet** (gRPC) — reserve/release funds
- **match-core-{market}** (gRPC) — submit orders to matching engine
- **ticker-query** (gRPC) — price validation
- **global-price-tracker** (gRPC) — external price reference
- **user-state-query** (gRPC) — user state checks (KYC, locks)
- **config-service** (Kafka consumer) — dynamic market config

## Kafka

- Consumes: `orderbook.match_price`, `config-service-events`
- Does NOT produce — match-core produces order events

## Key Details

- Stateless, horizontally scalable
- Match engine address templated: `match-{currency}-{payment}:50059`
- Uses Redis for idempotency, user cache, order cache
- Uses corekit + proto-hub
