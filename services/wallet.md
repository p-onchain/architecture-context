# wallet

> Core balance management service. Handles fund reservations, settlements, transfers, and commission processing for all trading and transaction operations.

## Quick Facts

- **Repo:** p-blackswan/wallet
- **Language:** Go
- **Port:** gRPC :50058
- **DB:** PostgreSQL (`wallet`)
- **Depends on:** Redis (commission rates, tx dedup)

## What It Does

1. Manages user balances (available, locked, blocked, credit)
2. Reserves funds for trades (called by order-api before matching)
3. Settles trades by consuming match events from Kafka
4. Processes deposits/withdrawals (called by transaction service)
5. Handles internal transfers, staking reserves, promotions
6. Produces ledger-logs to Kafka for downstream persistence

## gRPC Interface (WalletService)

Trade operations:
- `ReserveTradeFund` — lock funds before matching
- `ReleaseFund` — release reserved funds (cancel, deposit finalize)

Balance mutations:
- `AddAvailableFund` — fiat deposit
- `AddBlockedFund` — crypto deposit init (pending confirmation)
- `RemoveBlockedFund` — crypto deposit confirmed → available
- `Withdraw` — initiate withdrawal
- `FinalizeWithdraw` / `CancelFinalizeWithdraw` — complete or rollback withdrawal

Transfer operations:
- `InternalTransfer` — offchain/quick/bucket transfer
- `InstantInternalTransfer` — admin/staking instant transfer
- `LockedInternalTransfer` / `CancelLockedInternalTransfer`
- `BlockedInternalTransfer` / `CancelBlockedInternalTransfer`

Query:
- `GetUserAssetBalances` — get balances for a specific asset
- `StreamUserWallet` — stream all user balances (server-side streaming)

## Kafka

**Consumes:**
- `order.events.match.{market}` — settle trades (debit maker, credit taker)
- `order.events.status.{market}` — handle order status changes
- `user-commission-events` — commission rate updates

**Produces:**
- `ledger-logs` — every balance mutation as a ledger entry (consumed by wallet-ledger-sink and read-mono)
- `external.ledger-logs` — mirror for external consumers (via redpanda-connect)

## Redis Usage

- DB 2: Commission rates hash (`commission_rates` key, populated by commission-update-worker)
- DB 3: Transaction deduplication (TTL: 24h)

## Architecture Notes

- Wallet is the **source of truth** for balances — all mutations go through it
- Uses optimistic concurrency with PostgreSQL for balance updates
- Commission rates are cached in Redis, with default fallback (maker: 0.01, taker: 0.02)
- System accounts: user_id 42 (trade commissions), user_id 43 (withdraw commissions)
- Asset rehydration: can rebuild in-memory state from DB in batches (100k assets, 32 partitions)
- Depends on wallet-ledger-sink being healthy before starting (sync check via Kafka consumer lag)

## Related Services

- **wallet-ledger-sink** — persists ledger-logs from Kafka to PostgreSQL (the durable store)
- **wallet-validator** — validates wallet state consistency
- **wallet-asset-cleanup** — cleans up dust/zero-balance assets
- **balance-reservation-migration** — backfills balance_reservations table from ledger-logs
