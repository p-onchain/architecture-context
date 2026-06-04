# match-core (Matching Engine)

> Per-market order matching engine. In-memory order book, price-time priority matching.

- **Repo:** p-blackswan/match-core
- **Lang:** Go
- **Port:** gRPC :50059
- **One instance per market** (e.g., `match-btc-try`, `match-eth-try`)

## Talks To

- **order-api** (gRPC, inbound) — receives CreateOrder, CancelOrder, UpdateOrder
- **Redpanda** (produces) — match events, status events, orderbook state

## Kafka

- Produces: `order.events.match.{market}`, `order.events.status.{market}`, `orderbook.match_price`, `orderbook.state`
- Does NOT consume Kafka

## Key Details

- Stateful — order book in memory, persistence via Redpanda WAL (S3 snapshots planned but not yet implemented)
- No external DB dependency
- Market definitions in `p-blackswan/market-configs` (YAML, GitOps)
- ArgoCD generates one Deployment per market YAML
- Events use Protobuf + Schema Registry
