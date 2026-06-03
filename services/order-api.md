# order-api

> High-performance order entry gateway. Accepts orders from BFF, validates, reserves funds, and routes to the matching engine.

## Quick Facts

- **Repo:** p-blackswan/order-api
- **Language:** Go
- **Ports:** HTTP :9000, gRPC :60060
- **DB:** PostgreSQL (`order_api`)
- **Depends on:** wallet (gRPC), match-{market} (gRPC), config-service (Kafka), ticker (gRPC), global-price-tracker (gRPC), user-state (gRPC)

## What It Does

1. Receives order create/cancel/update requests via HTTP (from bff-api) or gRPC (from conditional-order)
2. Validates the order (price limits, market status, user state)
3. Calls `wallet.ReserveTradeFund()` to reserve user funds
4. Routes to the correct `match-{market}` instance via gRPC (address pattern: `match-{currency}-{payment}:50059`)
5. Returns synchronous response; match results arrive asynchronously via Kafka

## gRPC Downstream Calls

- `wallet:50058` — WalletService.ReserveTradeFund, ReleaseFund
- `match-{market}:50059` — OrderService.CreateOrder, CancelOrder, UpdateOrder
- `ticker:50073` — price validation
- `global-price-tracker:50079` — external price reference
- `user-state:50084` — user state checks

## Kafka

- **Consumes:** `orderbook.match_price` (for price validation), `config-service-events` (dynamic config)
- **Does NOT produce** to Kafka directly — the match engine produces order events

## Redis Usage

- DB 0: User cache
- DB 1: Idempotency keys (dedup order submissions)
- DB 5: User config cache (user-specific limits)
- DB 7: Order cache (shared with order-responder for fast cancel)

## Key Proto Contracts

- `proto/order/v1/service.proto` — OrderService (CreateOrder, CancelOrder, UpdateOrder, CancelOrders)
- `proto/order/v1/order.proto` — Order types, enums (OrderSide, OrderType, TimeInForce)

## Architecture Notes

- Stateless — horizontally scalable
- Uses corekit for DB, Kafka, gRPC client setup
- Match engine address is templated: `MATCH_SERVICE_ADDR_BASE=match-%s:50059` → format with `{currency}-{payment}`
- Fast cancel feature uses Redis order cache (DB 7) shared with order-responder
