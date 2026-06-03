# match (Matching Engine)

> Per-market order matching engine. Maintains in-memory order book, executes trades, produces events.

## Quick Facts

- **Repo:** p-blackswan/match
- **Language:** Go
- **Port:** gRPC :50059
- **DB:** None (stateful in-memory, snapshots to S3)
- **One instance per market** (e.g., `match-btc-try`, `match-eth-try`)

## What It Does

1. Receives orders from order-api via gRPC (CreateOrder, CancelOrder, UpdateOrder)
2. Maintains an in-memory order book (bids/asks)
3. Executes price-time priority matching
4. Produces match events and order status events to Kafka
5. Periodically snapshots order book state to S3 for recovery

## gRPC Interface

- `OrderService.CreateOrder` — add order to book, attempt match
- `OrderService.CancelOrder` — remove order from book
- `OrderService.CancelOrders` — batch cancel
- `OrderService.UpdateOrder` — modify existing order

## Kafka Production

- `order.events.match.{market}` — MatchEvent (contains matched taker/maker orders, match price, quantity)
- `order.events.status.{market}` — Order status updates (filled, partially filled, cancelled)
- `orderbook.match_price` — Last match price per market
- `orderbook.state` — Order book state snapshots for the orderbook projection

## Market Configuration

Markets are defined in `p-blackswan/market-configs` as YAML files:
```yaml
market:
  name: "btc-try"
  # currency, payment, precision, snapshot intervals, etc.
```

ArgoCD watches market-configs and generates one Deployment per enabled market YAML.

## Architecture Notes

- **Stateful** — order book lives in memory. Recovery via S3 snapshots + Kafka replay
- Each instance is single-threaded for the order book (no locking needed)
- Snapshot interval is configurable per market (default: 5m or every 1000 operations)
- Match events use Protobuf + Schema Registry serialization
- match-forge persists match events to PostgreSQL as an append-only event store
