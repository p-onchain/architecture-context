# Flow: Deposit & Withdrawal

> End-to-end flows for crypto and fiat deposits/withdrawals.

## Crypto Deposit

```
1. User gets deposit address (transaction service → custody-integration)
2. User sends crypto on-chain

3. onchain service (TypeScript) detects incoming transaction:
   → Calls transaction.CryptoDepositService.CreateUnconfirmedDeposit
   → transaction calls wallet.AddBlockedFund (funds visible but not spendable)
   → Produces: crypto_deposit_created_event

4. Blockchain confirmations accumulate:
   → onchain monitors confirmation count
   → When threshold met → transaction.VerifyDeposit
   → transaction calls wallet (move blocked → available)
   → Produces: crypto_deposit_verified_event

5. Compliance check:
   → elliptic-screener analyzes transaction (AML)
   → If flagged → manual review in gopanel

6. Read-side updates:
   → read-mono/balance-projection updates user balance view
   → read-mono/financial-history-projection adds deposit record
   → read-mono/transaction-projection updates transaction status
   → Notification sent to user (push/email)
```

## Crypto Withdrawal

```
1. Client → bff → transaction.CryptoWithdrawService.CreateWithdraw
   → MFA verification (transaction.TransactionMfaService)
   → Elliptic AML screening
   → Produces: crypto_transaction_created_event

2. Fund reservation:
   → transaction calls wallet.Withdraw (locks funds)

3. Custody signing:
   → transaction → custody-integration → HSM signs transaction
   → Produces: crypto_transaction_pending_signature_event
   → After signing: crypto_transaction_signed_event

4. Broadcast to blockchain:
   → custody-integration broadcasts signed tx
   → onchain monitors for confirmation

5. Confirmation:
   → onchain detects confirmation
   → transaction.VerifyCryptoTransaction
   → wallet.FinalizeWithdraw (deducts funds permanently)
   → Produces: crypto_transaction_verified_event

6. Failure handling:
   → If broadcast fails or times out:
     → transaction.CancelWithdraw
     → wallet.CancelFinalizeWithdraw (release funds back)
     → Produces: transaction_failed_event
```

## Fiat Deposit

```
1. Bank transfer arrives:
   → bank-integration receives bank notification (webhook/polling)
   → Calls transaction.FiatDepositService.CreateFiatDeposit
   → transaction calls wallet.AddAvailableFund (immediately available)
   → Produces: fiat_deposit_created_event

2. Instant deposit (alternative):
   → Client → bff → transaction.CreateUnverifiedFiatInstantDeposit
   → Bank confirms → transaction.VerifyFiatInstantDeposit
   → wallet.AddAvailableFund
   → Produces: fiat_instant_deposit_verified_event
```

## Fiat Withdrawal

```
1. Client → bff → transaction.FiatWithdrawService.CreateWithdraw
   → MFA verification
   → Compliance checks

2. Fund reservation:
   → wallet.Withdraw (locks funds)
   → Produces: fiat_transaction_created_event

3. Bank processing:
   → transaction → bank-integration (sends to bank API)
   → transaction.StartTransaction
   → Produces: transaction_started_event

4. Completion:
   → Bank confirms transfer
   → bank-integration notifies transaction
   → wallet.FinalizeWithdraw
   → Produces: fiat_transaction_verified_event

5. Failure:
   → Bank rejects → transaction.MarkFailTransaction
   → wallet.CancelFinalizeWithdraw (release funds)
   → Produces: transaction_failed_event
```

## Services Involved

transaction (orchestrator), wallet (balances), onchain (blockchain monitoring), custody-integration (HSM signing), bank-integration (fiat), elliptic-screener (AML), notification (user alerts)

Event protos: `proto/transaction/event/v1/` in proto-hub
