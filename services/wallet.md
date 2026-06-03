# wallet

> Core balance management. Source of truth for all user balances. Handles fund reservation, trade settlement, deposits, withdrawals, transfers.

- **Repo:** p-blackswan/wallet
- **Lang:** Go
- **Port:** gRPC :50058
- **DB:** PostgreSQL

## Talks To

- **order-api** (gRPC, inbound) — fund reservation for trades
- **transaction** (gRPC, inbound) — deposit/withdraw balance mutations
- **Redis** — commission rates cache, transaction dedup

## Kafka

- Consumes: `order.events.match.{market}`, `order.events.status.{market}`, `user-commission-events`
- Produces: `ledger-logs` (every balance mutation → consumed by wallet-ledger-sink and read-mono)

## Key Details

- All balance mutations go through wallet — it's the single source of truth
- wallet-ledger-sink persists ledger-logs to PostgreSQL
- wallet-validator checks consistency
- System accounts: user_id 42 (trade commissions), user_id 43 (withdraw commissions)
