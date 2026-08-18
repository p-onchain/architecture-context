# Flow: Order Lifecycle

> End-to-end flow of a trade order from client submission to settlement.
> **Last verified 2026-08-17.**

## 1. Order Submission

```
Client (micro-web / Samaritan)
  → Cloudflare
    → Envoy Gateway (external-gw, ns gateway-api-system)
      → KrakenD (service.name=krand; JWT validation, rate limiting)
        → bff-api :3000 (web) / bff-client :8080 (mobile)
          → order-api (HTTP POST /order — singular; header X-User-ID must equal body user_id)
```

> Market is `currency` + `payment` as separate lowercase fields, not a combined symbol.
> Stop/OCO use `POST /conditional-order` and `POST /oco-order`. Full contract in `services/order-api.md`.

## 2. Validation & Fund Reservation (order-api)

```
order-api:
  1. Validate request (market exists, price within limits, user not locked)
     - Min-order-value floor snapped to nearest achievable step (EXCH-2698)
  2. Check idempotency key (Redis DB 1) — reject duplicates
  3. Fetch commission rates from Redis (via wallet)
  4. Call wallet.ReserveTradeFund(gRPC) — locks user's funds
     - Buy order: locks payment currency (e.g., TRY)
     - Sell order: locks base currency (e.g., BTC)
  5. If reservation fails → return error (insufficient balance)
  6. Store order in Redis cache (DB 7) for fast-cancel
  7. Route to match engine
```

## 3. Match Engine Routing (order-api → match)

```
order-api:
  1. Determine target: match-{currency}-{payment}:50051
     (e.g., "btc-try" → match-btc-try:50051; via MATCH_SERVICE_ADDR_BASE=match-%s:50051)
  2. Call OrderService.CreateOrder(gRPC)
  3. Return order_id to client (async result will follow)
```

## 4. Matching (match engine)

```
match-{market}:
  1. Insert order into in-memory order book
  2. Run price-time priority matching
  3. For each match:
     - Create MatchEvent (taker, maker, price, quantity, match_id)
  4. Produce to the INTERNAL REDPANDA cluster (shared, un-suffixed topics —
     consumers filter by market from payload):
     - order.events.match → MatchEvent (one or more matches)
     - order.events.status → OrderStatus (filled/partial/open)
  5. Update orderbook state → orderbook.state topic
  6. Update last price → orderbook.match_price topic (only when a match occurs; empty book = no price)
```

⚠️ **Broker boundary.** Everything above is on the internal Redpanda cluster. `redpanda-connect`
mirrors `order.events.match`, `order.events.status` and `ledger-logs` to **MSK**, which is where the
read-side projections consume them — so read-side lag has two hops, and any offset-reset rebuild is
bounded by **MSK** retention. Only `orderbook-projection` and `orderbook-wapi-projection` read
Redpanda directly.

## 5. Settlement (wallet — Kafka consumer)

```
wallet (consumes order.events.match):
  For each MatchEvent:
    1. Debit maker's reserved funds
    2. Credit taker with bought asset (minus commission)
    3. Credit maker with received payment (minus commission)
    4. Transfer commissions to system accounts (user 42)
    5. Produce ledger-logs to Kafka (every balance mutation)
```

## 6. Persistence

```
Parallel consumers:
  - wallet-ledger-sink: ledger-logs → PostgreSQL (durable balance history)
  - match-forge: order.events.match → PostgreSQL (append-only trade store)
  - read-mono projections (all on MSK unless noted):
    - open-order-projection: tracks open orders (ns saul)
    - financial-history-projection: builds trade history (ns saul; also consumes `commission`, `transaction-events`)
    - ticker-projection: updates 24h stats (ns saul, ClickHouse) — also consumes orderbook.bestbidask
    - klines-projection: builds OHLCV candles (ns saul, ClickHouse)
    - uservolume-projection / cost-basis-projection (ns saul) — cost-basis is CEX-only
    - balance-projection: read-side balances (ns BLACKSWAN, batched; DLQ `balance-projection.dlq`)
    - orderbook-projection + orderbook-wapi-projection: order book snapshots
      (ns saul, **direct off Redpanda**; WAPI stream has its own Redis key schema in `rediskeys`)
    - ws-projection (ns corleone) → `websocket-events` → ws-hub
```

> **pnl-projection is not a match consumer.** It builds P&L from `ledger-logs` (wallet) plus
> `ticker.daily` / `ticker.last24h`, and — for DEX — `pnl.defi.asset.updates` from the read-mono
> `defi` domain. See `services/read-mono.md`.

## 7. Response to Client

```
order-responder (consumes order.events.match + order.events.status):
  1. Aggregates match results for the order
  2. Writes extended response to Redis order cache (DB 7)

order-api (extended response path):
  3. Polls Redis (match_response_repository) for the aggregated result
  4. Returns to BFF / client once available (or timeout)

⚠️  Previously order-api consumed match-response directly from Kafka.
    As of EXCH-6786 (2026-05), that Kafka consumer was removed.
    All match-response aggregation now goes through order-responder → Redis.

ws-hub / wapi:
  - Streams real-time updates to connected clients
  - Order status, trade executions, balance changes, ticker updates
```

## Cancel Flow

```
Client → bff → order-api.CancelOrder
  → Check Redis order cache (DB 7) for fast cancel
  → Call match-{market}.CancelOrder(gRPC)
  → Match engine removes from order book
  → Produces order.events.status (cancelled)
  → wallet releases reserved funds (ReleaseFund)
```

## Conditional Order Flow

```
Client → bff → conditional-order.CreateConditionalOrder(gRPC :50051)
  → Stored in PostgreSQL
  → conditional-order consumes orderbook.match_price
  → When trigger price hit:
    → Calls order-api gRPC :50051 (OrderAPIConditionalOrderService.Execute callback)
    → Normal order flow continues
```
