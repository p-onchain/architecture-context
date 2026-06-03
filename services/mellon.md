# mellon (Auth Service)

> Authentication service — Monosign replacement. Handles OIDC, session management, and token issuance.

## Quick Facts

- **Repo:** p-blackswan/mellon
- **Language:** Go
- **Named after:** "Speak, friend, and enter" (Lord of the Rings)
- **Depends on:** user-service, Redis (sessions)

## What It Does

1. OIDC-compliant authentication (login, token refresh, logout)
2. Session management
3. JWT token issuance (consumed by KrakenD for request validation)
4. MFA flow orchestration
5. Passkey (WebAuthn) authentication
6. Device trust (DTT) management

## Architecture Notes

- KrakenD validates JWTs issued by mellon on every request
- Sessions stored in Redis
- Integrates with user-service for user lookup and security settings
- krakend-revoke-server handles token revocation propagation to KrakenD
- revoke-relay assists with session invalidation across clusters
