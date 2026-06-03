# transaction

> Deposit & withdraw operations for both crypto and fiat. Manages the full lifecycle of blockchain and bank transactions.

## Quick Facts

- **Repo:** p-blackswan/transaction
- **Language:** Go
- **Port:** gRPC :50076
- **DB:** PostgreSQL
- **Depends on:** wallet (gRPC), custody-integration (gRPC), bank-integration (gRPC), user-service (gRPC), elliptic-screener

## What It Does

### Crypto Deposits
1. Receives blockchain deposit notifications (from onchain service)
2. Creates unconfirmed deposit → calls wallet.AddBlockedFund
3. After blockchain confirmation → calls wallet to move from blocked to available
4. Travel rule declaration for compliance

### Crypto Withdrawals
1. User initiates withdrawal via BFF
2. MFA verification
3. Elliptic AML screening
4. Calls wallet.Withdraw to reserve funds
5. Sends to custody for signing
6. Monitors blockchain for confirmation
7. Finalizes or rolls back via wallet

### Fiat Deposits
1. Bank notification or instant deposit trigger
2. Creates deposit record
3. Calls wallet.AddAvailableFund

### Fiat Withdrawals
1. User initiates via BFF
2. MFA + compliance checks
3. Calls wallet.Withdraw
4. Sends to bank integration
5. Bank confirmation → finalize

## gRPC Interface

- `CryptoDepositService` — CreateUnconfirmedDeposit, VerifyDeposit, CreateTravelRuleDeclaration, Admin ops
- `CryptoWithdrawService` — CreateWithdraw, CancelWithdraw, sign/verify flow
- `FiatDepositService` — CreateFiatDeposit, instant deposit flow
- `FiatWithdrawService` — CreateWithdraw, CancelWithdraw, StartTransaction, MarkFail, Complete
- `TransactionMfaService` — MFA verification for withdrawals
- `TransactionFraudService` — Fraud checks

## Kafka Production

Produces extensive transaction lifecycle events:
- `crypto_deposit_created`, `crypto_deposit_verified`
- `crypto_transaction_created`, `crypto_transaction_signed`, `crypto_transaction_verified`
- `fiat_deposit_created`, `fiat_transaction_created`, `fiat_transaction_verified`
- `transaction_started`, `transaction_canceled`, `transaction_failed`
- `staking_reward_deposited`

All events defined in `proto/transaction/event/v1/`.

## Deployed Components

- **transaction** — main command service
- **transaction-query** — read model (via read-mono)
- **transaction-worker** — async processing (blockchain monitoring, retries)
- **transaction-job** — scheduled jobs (reconciliation, timeout handling)
- **transaction-ledger-outbox** — outbox pattern for ledger events

## Related Services

- **onchain** (p-blackswan/onchain, TypeScript) — blockchain monitoring, deposit detection
- **custody-integration** — HSM/custody signing bridge
- **bank-integration** — bank API integration for fiat
- **elliptic-screener** — AML/compliance screening
