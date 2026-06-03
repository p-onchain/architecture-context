# user-service

> User management service. Handles user CRUD, KYC lifecycle, security settings, segments, locks, consents, and contracts.

## Quick Facts

- **Repo:** p-blackswan/user-service
- **Language:** Go
- **Port:** gRPC :50074
- **DB:** PostgreSQL
- **Depends on:** mellon (auth), notification (events)

## What It Does

1. User registration and profile management
2. KYC lifecycle (unverified → verified → banned, with investigation states)
3. Security settings (MFA, password, passkeys)
4. User segmentation (green, green+, black, black+, paribull)
5. User locks (trade lock, withdraw lock, etc.)
6. Consent management (KVKK, marketing, etc.)
7. Contract signing (user agreements)
8. Favourites and feedback
9. Loyalty transfers
10. Sanctions/compliance profiles

## gRPC Interface (from proto-hub)

**Command Services:**
- `UserCommandService` — create, update personal details, change email/mobile/password
- `KycCommandService` — verify, reject, ban, set investigation status
- `SecurityCommandService` — enable/disable MFA, passkey management
- `SegmentCommandService` — update user segment
- `LockCommandService` — create/delete user locks
- `ConsentCommandService` — update user consents
- `ContractCommandService` — create/update/sign contracts
- `FavoritesCommandService` — toggle favourite markets
- `FeedbackCommandService` — create user feedback
- `NotesCommandService` — create admin notes on users
- `SanctionsCommandService` — manage compliance profiles
- `LoyaltyTransferCommandService` — loyalty point transfers
- `PasskeyService` — WebAuthn passkey registration/auth

**Query Services:**
- `UserQueryService` — get user by ID, search users
- `ConsentQueryService` — get user consents
- `SegmentQueryService` — get user segment info
- `FavoritesQueryService` — get user favourites
- `FeedbackQueryService` — get user feedback
- `LoyaltyTransferQueryService` — query loyalty transfers

## Kafka Production

Produces extensive user lifecycle events to Kafka (all in `proto/user/v1/event/`):
- User created, details changed, email/mobile changed
- KYC status changes (verified, rejected, banned, under investigation)
- MFA enabled/disabled, password changed
- Segment changes, lock changes
- Consent changes, contract signing
- Favourite toggled, feedback created

## Deployed Components

- **user-service** — main command service
- **user-query** — read model (via read-mono)
- **user-state-projection** / **user-state-query** — aggregated user state (KYC + locks + segment in one view)
- **user-worker** — async processing
- **user-jobs** — scheduled jobs

## Architecture Notes

- User IDs are UUIDv7 format
- User state projection aggregates data from multiple sources into a single queryable view
- Segment determines commission rates (via commission system)
