# user-service

> User management. CRUD, KYC lifecycle, security settings, segments, locks, consents, favourites, loyalty, sanctions.

- **Repo:** p-blackswan/user-service
- **Lang:** Go
- **Port:** gRPC :50074
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
- Deployed as: user-service (command), user-query + user-state-query (read-mono)
- User IDs are UUIDv7
- Segment determines commission rates
