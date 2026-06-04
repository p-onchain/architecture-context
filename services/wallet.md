# wallet

> Core balance management. Source of truth for all user balances. Handles fund reservation, trade settlement, deposits, withdrawals, transfers.

- **Repo:** p-blackswan/wallet
- **Lang:** Go
- **Port:** gRPC **:50051**  (namespace `blackswan`)
- **DB:** PostgreSQL

## Talks To

- **order-api** (gRPC, inbound) — fund reservation for trades (`ReserveTradeFund`, `ReleaseFund`)
- **transaction** (gRPC, inbound) — deposit/withdraw balance mutations
- **Redis** — commission rates cache, transaction dedup

## Kafka

- Consumes: `order.events.match`, `order.events.status` (shared, payload-filtered), `user-commission-events`, `config-service-events`
- Produces: `ledger-logs` (every balance mutation → consumed by wallet-ledger-sink and read-mono)

## Key Details

- All balance mutations go through wallet — it's the single source of truth
- **gRPC has NO auth interceptor** (only panic-recovery), no NetworkPolicy/mTLS — any in-cluster client reaching `wallet:50051` can call it. Gating is at the network/edge (KrakenD), not per-method.
- Key methods: `ReserveTradeFund` (RESERVE_TRADE=10) / `ReleaseFund` (RELEASE_TRADE=11) for trades; `AddAvailableFund` credits available balance and accepts **only** `DEPOSIT_FIAT`(=2) (used for both fiat and crypto direct credits); `GetUserAssetBalances` reads balances. (`DEPOSIT_CRYPTO_FINALIZE`=1 is a `ReleaseFund` op, not AddAvailableFund.)
- wallet-ledger-sink persists ledger-logs to PostgreSQL; wallet-validator checks consistency
- System accounts: user_id 42 (trade commissions), user_id 43 (withdraw commissions)
