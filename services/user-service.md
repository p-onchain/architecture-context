# user-service

> User management. CRUD, KYC lifecycle, security settings, segments, locks, consents, favourites, loyalty, sanctions.

> **Cluster placement verified 2026-08-17**; the module/ownership detail below dates from 2026-06.

- **Repo:** p-blackswan/user-service
- **Lang:** Go
- **Port:** gRPC **:50051**  (namespace **`shelby`**, not `blackswan`)
- **⚠️ Runs on Huawei Cloud in prod** — cluster **`exc-prod-hw`** (`.cd/helm/hw-prod/`), together
  with `user-worker`, `user-jobs`, `tier-worker` and read-mono's `user-query`. Only the derived
  `user-state-projection` / `user-state-query` run on AWS (`exc-prod-alpha`).
- **DB:** PostgreSQL
- **Kafka:** MSK — produces `user-events`

## Talks To

- **mellon** (inbound, auth context) — user auth context
- **notification** (Kafka) — user event notifications
- **read-mono** (Kafka) — user projections consume events

## Kafka

- Produces: user lifecycle events (created, KYC changes, MFA, segment, lock, consent, favourites, etc.)
- All event protos in `proto/user/v1/event/`

## Key Details

- Modules: customer, favorites, feedback, lock, loyalty, note, sanctions, security, contract, outbox
- Deployed as: user-service (command) + user-worker/user-jobs/tier-worker on `exc-prod-hw`;
  user-query on `exc-prod-hw`; user-state-projection/query on `exc-prod-alpha` — namespace `shelby` everywhere
- ⚠️ read-mono's `favourite-query` and `feedback-query` were **decommissioned** (E2B-97) — those reads
  no longer have a CQRS query service
- `GetBulkUsers` is the enumeration RPC the read side leans on (`corekit/userquery`); a 45-pod
  rollout hitting it without jitter has caused a stampede, so treat it as a shared, rate-sensitive API
- **User IDs are UUIDv7, generated server-side on every create path** — callers cannot supply a chosen UUID via any API
- New users start `kyc_status=unverified`; **no test-env KYC bypass** (real `pkyc` provider even in test)
- **`CanTrade` is NOT owned here** — it's computed in read-mono's `user-state` projection (active + KYC verified + contract signed + fiat deposit + no market/fraudbank lock), emitted on Kafka `user-state-events`, enforced in order-api. user-service only emits the raw inputs (KYC status, locks via LockCommandService, status, contract).
- Segment determines commission rates
