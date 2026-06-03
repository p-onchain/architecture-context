# Flow: Order Lifecycle

> End-to-end flow of a trade order from client submission to settlement.

## 1. Order Submission

```
Client (Samaritan/Web)
  → KrakenD (JWT validation)
    → bff-api/bff-client (HTTP)
      → order-api (HTTP POST /v1/orders)
```

## 2. Validation & Fund Reservation (order-api)

```
order-api:
  1. Validate request (market exists, price within limits, user not locked)
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
  1. Determine target: match-{currency}-{payment}:50059
     (e.g., "btc-try" → match-btc-try:50059)
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
  4. Produce to Kafka:
     - order.events.match.{market} → MatchEvent (one or more matches)
     - order.events.status.{market} → OrderStatus (filled/partial/open)
  5. Update orderbook state → orderbook.state topic
  6. Update last price → orderbook.match_price topic
```

## 5. Settlement (wallet — Kafka consumer)

```
wallet (consumes order.events.match.{market}):
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
  - match-forge: order.events.match.{market} → PostgreSQL (append-only trade store)
  - read-mono projections:
    - open-order-projection: tracks open orders
    - financial-history-projection: builds trade history
    - ticker-projection: updates 24h stats
    - klines-projection: builds OHLCV candles
    - balance-projection: updates read-side balances
    - pnl-projection: calculates P&L
    - orderbook-projection: rebuilds order book snapshots
```

## 7. Response to Client

```
order-responder (consumes order.events.match + order.events.status):
  1. Aggregates match results for the order
  2. Writes to Redis order cache (DB 7)
  3. BFF can poll or client receives via WebSocket

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
Client → bff → conditional-order.CreateConditionalOrder(gRPC :50052)
  → Stored in PostgreSQL
  → conditional-order consumes orderbook.match_price
  → When trigger price hit:
    → Calls order-api.CreateOrder(gRPC :60060)
    → Normal order flow continues
```

## Key Design Decisions

- **Async settlement:** order-api returns immediately after submitting to match engine. Settlement happens via Kafka consumers.
- **Exactly-once semantics:** wallet uses transaction deduplication (Redis DB 3, 24h TTL) to prevent double-settlement.
- **Fast cancel:** Redis order cache (DB 7) shared between order-api and order-responder enables cancel without DB lookup.
- **Per-market topics:** each market has its own Kafka topics for match/status events, enabling independent scaling.
