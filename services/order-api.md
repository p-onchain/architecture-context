# order-api

> Order entry gateway. Accepts orders, validates, reserves funds via wallet, routes to matching engine.

- **Repo:** p-blackswan/order-api
- **Lang:** Go
- **Ports (live cluster):** HTTP **:3000**, gRPC **:50051**  (repo container defaults are HTTP 9000 / gRPC 60060; the deployment/Service expose 3000/50051). gRPC server only serves the conditional-order `Execute` callback — order creation is HTTP-only.

## HTTP order contract (ground truth)

- Order create: `POST /order` (singular). Header `X-User-ID: <uuid>` — must equal body `user_id` or returns 403. Cancel: `DELETE /order?order_id=&currency=&payment=`.
- Body fields: `user_id, currency, payment, price, quantity, order_side, order_type, time_in_force?, total_price?`. Market = separate `currency`+`payment` (lowercase, e.g. `btc`/`try`), NOT a combined symbol.
- Enums: `order_side ∈ {ORDER_SIDE_BID, ORDER_SIDE_ASK}`, `order_type ∈ {ORDER_TYPE_LIMIT, ORDER_TYPE_MARKET, ORDER_TYPE_LIMIT_MAKER}`, `time_in_force ∈ {GTC, IOC, FOK}` (market orders must be GTC). Stop/OCO use separate routes (`POST /conditional-order`, `POST /oco-order`).

## Talks To

- **wallet** (gRPC `wallet:50051`) — reserve (`ReserveTradeFund`) / release (`ReleaseFund`) funds
- **match-{currency}-{payment}** (gRPC `:50051`) — submit orders to matching engine (`OrderService.CreateOrder`)
- **ticker-query** (gRPC, namespace `saul`) — price validation
- **global-price-tracker-api** (gRPC, namespace `saul`) — external price reference
- **user-state-query** (gRPC `user-state-query.shelby:50051`) — `CanTrade` / user state checks
- **config-service** (REST `config-service.shelby:8080` + Kafka) — market config

## Kafka

- Consumes: `orderbook.match_price`, `config-service-events`, `user-state-events`
- Does NOT produce order events — match produces them

## Key Details

- Stateless, horizontally scalable
- Match engine address templated via `MATCH_SERVICE_ADDR_BASE=match-%s:50051` where `%s` = `{currency}-{payment}` → e.g. `match-btc-try:50051`
- Market config fetched via REST `GET {CONFIG_URL}/default` (+ `/merged/{userID}`) and refreshed via Kafka `config-service-events`
- **No rate limiting in order-api** — rate limiting lives only at KrakenD (in-cluster callers to `order-api:3000` bypass it entirely)
- `CanTrade` is read from user-state-query (computed in read-mono's user-state projection), cached in Redis; unknown users default to `CanTrade=false` → 403
- Uses Redis for idempotency, user cache, order cache; corekit + proto-hub
