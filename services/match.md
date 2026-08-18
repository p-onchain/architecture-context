# match (Matching Engine)

> Per-market order matching engine. In-memory order book, price-time priority matching.

> **Cluster/broker facts verified 2026-08-17** (live prod pods + `market-configs`); the HTTP/gRPC contract details below date from 2026-06.

- **Repo:** p-blackswan/match  (NOTE: `p-blackswan/match-core` is **ARCHIVED** — `match` is the active repo)
- **Lang:** Go
- **Port:** gRPC **:50051** in prod/test cluster (repo/local default is `GRPC_PORT=50059`; deployment overrides to 50051)
- **gRPC service:** `OrderService` (CreateOrder, CancelOrder, CancelOrders, UpdateOrder); health service name `grpc.health.v1.MatchEngine`
- **One Service per market** (e.g., `match-btc-try:50051`, `match-eth-try:50051`); namespace `blackswan`.
  **263 market apps in prod** as of 2026-08-17, generated from `p-blackswan/market-configs` by its own
  ApplicationSet (`platform-gitops/app-of-apps/exchange/apps-market-configs.yaml`) — the engines are
  the bulk of the ~572 pods in that namespace.
- **Market config:** loaded over HTTP from config-service (`CONFIG_URL=.../default`), NOT a YAML mount. Per-market YAML in `p-blackswan/market-configs` only carries deployment identity (CURRENCY/PAYMENT/MARKET), no precision/price.

## Talks To

- **order-api** (gRPC, inbound) — receives CreateOrder, CancelOrder, UpdateOrder
- **Redpanda** (produces) — match events, status events, orderbook state

## Kafka

**Broker: the internal Redpanda cluster** (`10.240.65.47,10.240.130.74,10.240.252.225,10.240.104.25,10.240.178.238:9092`,
plaintext, in-VPC, not a K8s workload) — *not* MSK. Read-side consumers see these topics only after
`redpanda-connect` mirrors them to MSK, so their retention/replay budget is MSK's, not Redpanda's.

- Produces (all **shared, un-suffixed** — consumers filter by market from payload): `order.events.match`, `order.events.status`, `orderbook.match_price`, `orderbook.state`
- Produces WAL (per-market suffixed): `order.requests.{market}` (e.g. `order.requests.btc-try`)
- Consumes: its own WAL `order.requests.{market}` on startup (crash recovery) + `config-service-events`
  (live market-config refresh — arrives on Redpanda via the MSK→Redpanda leg of the bridge)
- Schema Registry: the Confluent SR in ns `blackswan` (`schema-registry-{0,1,2}…:8081`)

## Key Details

- Stateful — order book in memory; durability via S3 snapshots (`orderbook-snapshots` bucket) + Kafka WAL replay on boot
- A fresh/empty market has **no price** until the first trade — `orderbook.match_price` is published only when a match occurs. There is **no seed/initial price** anywhere (not in match, config-service, or market-configs)
- No external DB dependency (uses Redis for idempotency, S3 for snapshots)
- Market definitions in `p-blackswan/market-configs` (YAML, GitOps); ArgoCD generates one Deployment per market YAML. Trading params (precision/steps) come from config-service, not the YAML
- Events use Protobuf + Schema Registry; decimals use `govalues/decimal`
- **client_order_id propagation (EXCH-6672, 2026-06):** `ExtraData.ClientOrderId` is now carried through from `CreateOrderRequest` → domain command → match events. Validation is intentionally **absent in match** — match passes the value through without validating format/uniqueness (validation lives at order-api / upstream). proto-hub bumped to v1.23.23.
