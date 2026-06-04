# user-service

> User management. CRUD, KYC lifecycle, security settings, segments, locks, consents, favourites, loyalty, sanctions.

- **Repo:** p-blackswan/user-service
- **Lang:** Go
- **Port:** gRPC **:50051**  (namespace **`shelby`**, not `blackswan`)
- **DB:** PostgreSQL

## Talks To

- **mellon** (inbound, auth context) — user auth context
- **notification** (Kafka) — user event notifications
- **read-mono** (Kafka) — user projections consume events

## Kafka

- Produces: user lifecycle events (created, KYC changes, MFA, segment, lock, consent, favourites, etc.)
- All event protos in `proto/user/v1/event/`

## Key Details

- Modules: customer, favorites, feedback, lock, loyalty, note, sanctions, security, contract, outbox
- Deployed as: user-service (command), user-query + user-state-query (read-mono) — all in namespace `shelby`
- **User IDs are UUIDv7, generated server-side on every create path** — callers cannot supply a chosen UUID via any API
- New users start `kyc_status=unverified`; **no test-env KYC bypass** (real `pkyc` provider even in test)
- **`CanTrade` is NOT owned here** — it's computed in read-mono's `user-state` projection (active + KYC verified + contract signed + fiat deposit + no market/fraudbank lock), emitted on Kafka `user-state-events`, enforced in order-api. user-service only emits the raw inputs (KYC status, locks via LockCommandService, status, contract).
- Segment determines commission rates
