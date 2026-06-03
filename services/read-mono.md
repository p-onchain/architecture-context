# read-mono

> CQRS read-side monorepo. Contains ~25 projection/query service pairs that build read-optimized views from Kafka events.

## Quick Facts

- **Repo:** p-blackswan/read-mono
- **Language:** Go (monorepo with go.work)
- **Architecture:** Each domain has a `projection` (Kafka consumer → DB writer) and `query` (gRPC server)
- **DB:** PostgreSQL (per-domain schemas), ClickHouse (analytics/heavy queries)

## Structure

```
read-mono/
├── libs/                    # Shared libraries (adapters, cache, DB, gRPC, messaging, partition, workerpool)
├── services/
│   ├── alarm/              # Price alarms
│   │   ├── projection/     # Consumes alarm events → PostgreSQL
│   │   └── query/          # gRPC AlarmQueryService
│   ├── balance/            # User balances
│   ├── campaign/           # Campaign projections
│   ├── commission/         # Commission rates/segments
│   ├── cost-basis/         # Cost basis tracking
│   ├── favourite/          # User favourite markets
│   ├── feedback/           # User feedback
│   ├── financial-history/  # Trade/deposit/withdraw history
│   ├── klines/             # OHLCV candlestick data
│   ├── open-order/         # Currently open orders
│   ├── orderbook/          # Order book snapshots
│   ├── orderbook-wapi/     # Order book for WAPI (optimized format)
│   ├── pnl/                # Profit & loss calculations
│   ├── pnl-agg/            # Aggregated PnL
│   ├── staking/            # Staking pools/stakes
│   ├── ticker/             # Market ticker (24h stats, last price)
│   ├── transaction/        # Transaction history
│   ├── user/               # User profile projections
│   ├── user-state/         # User state (KYC status, locks, etc.)
│   ├── uservolume/         # User trading volume
│   └── ws/                 # WebSocket projection (feeds ws-hub)
```

## How It Works

1. **Projection** services consume Kafka topics and build read-optimized views in PostgreSQL or ClickHouse
2. **Query** services expose gRPC APIs that BFF services call to serve read requests
3. Each projection maintains its own consumer group and can be scaled independently
4. Projections are idempotent — can be rebuilt by resetting consumer offsets

## Deployment

Each domain deploys as separate pods:
- `{domain}-projection` — Kafka consumer (Deployment)
- `{domain}-query` — gRPC server (Deployment + Service)

Some domains also have workers (e.g., `ticker-worker`, `pnl-worker`).

## Development Modes

- **Standalone** — service runs with its own Redpanda instance, events produced manually
- **Platform mode** — connects to platform's Redpanda cluster for full integration

## Architecture Notes

- Single generic Dockerfile (`.cd/docker/Dockerfile`) for all services — no service-specific Dockerfiles
- Shared libs in `libs/` are versioned on GitHub
- Uses corekit for Kafka consumers, DB connections, gRPC setup
- ClickHouse used for heavy analytical queries (financial-history, klines)
