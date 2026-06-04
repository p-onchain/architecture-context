# mellon (Auth Service)

> Authentication & identity service. OIDC, OAuth2, sessions, MFA, passkeys, SSO, API keys, and more.

- **Repo:** p-blackswan/mellon
- **Lang:** Go
- **Named after:** "Speak, friend, and enter" (LotR)

## Talks To

- **user-service** (gRPC) — user lookup, security settings
- **Redis** — session storage, caches
- **Kafka** — audit event publishing
- **KrakenD** — validates JWTs issued by mellon

## Capabilities

Auth/identity modules (each has its own package in `internal/`):
- OIDC, OAuth2, session, JWT issuance
- MFA, passkey (WebAuthn), device trust
- SSO, partner consent, API key management
- Registration, impersonation, admin user mgmt
- Audit logging, captcha

## Key Details

- krakend-revoke-server handles token revocation propagation
- revoke-relay handles cross-cluster session invalidation
- Clone the repo — `internal/` is well-organized, each module is self-contained
