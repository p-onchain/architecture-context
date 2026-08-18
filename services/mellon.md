# mellon (Auth Service)

> Authentication & identity service. OIDC, OAuth2, sessions, MFA, passkeys, SSO, API keys, and more.
> Replaced Monosign.

> **Deployment facts verified 2026-08-17**; capability list below dates from 2026-06 — clone `internal/` for the current module set.

- **Repo:** p-blackswan/mellon
- **Deployed as:** **`auth-service`** (ns `shelby`, exc-prod-alpha; 3 pods) — the *repo* named
  `auth-service` is **archived**, don't look for code there. Internal route `auth.int.paribu.com`.
- **Lang:** Go
- **Kafka:** MSK — produces `auth-audit-events`
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
- ⚠️ **revoke-relay is archived** — cross-cluster session invalidation no longer runs through it
- Passkey/WebAuthn SDKs live in p-utilities (`passkeys`, `passkey`) and p-onchain (`passkey`)
- KYC is **not** here: it moved to `estel` (Rust — `kyc-api`, `kyc-orchestrator`, `kyc-backoffice`),
  with `p-utilities/p-kyc` + `nitro-kyc` on the client side
- Clone the repo — `internal/` is well-organized, each module is self-contained
