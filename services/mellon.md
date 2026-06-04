# mellon (Auth Service)

> Authentication & identity service — Monosign replacement. Handles OIDC, OAuth2, session management, token issuance, registration, and more.

## Quick Facts

- **Repo:** p-blackswan/mellon
- **Language:** Go
- **Named after:** "Speak, friend, and enter" (Lord of the Rings)
- **Depends on:** user-service (gRPC), Redis (sessions/cache), Kafka (audit events)

## What It Does

1. **OIDC-compliant authentication** — login, token refresh, logout, discovery endpoint
2. **OAuth2 authorization code flow** — for partner/third-party app integrations
3. **Session management** — Redis-backed sessions with TTL and device binding
4. **JWT token issuance** — consumed by KrakenD for request validation
5. **MFA flow orchestration** — multi-factor authentication (TOTP, SMS, etc.)
6. **Passkey (WebAuthn)** — FIDO2/WebAuthn passkey registration and authentication
7. **Device trust (DTT)** — device trust token management for mobile
8. **SSO** — single sign-on for partner integrations
9. **Partner consent** — consent management for third-party data sharing
10. **API key management** — create, revoke, list API keys (user + admin endpoints)
11. **Admin user management** — admin-facing user lookup/management endpoints
12. **User registration** — new user registration flows with step tracking
13. **Impersonation** — admin impersonate-user capability
14. **Audit logging** — all auth events published to Kafka for audit trail
15. **Captcha verification** — captcha integration for bot protection

## Internal Modules

```
mellon/internal/
├── adapters/        # External service adapters
├── admin/           # Admin user management endpoints
├── apikey/          # API key CRUD (user + admin handlers)
├── app/             # Application bootstrap
├── audit/           # Kafka-based audit event publishing
├── captcha/         # Captcha verification
├── compat/          # Backward compatibility layer
├── crypto/          # Cryptographic utilities
├── device/          # Device trust management
├── domain/          # Core domain models
├── health/          # Health check endpoints
├── impersonate/     # Admin impersonation
├── mfa/             # Multi-factor authentication
├── middleware/       # HTTP middleware (auth, logging, etc.)
├── oauth2/          # OAuth2 authorization code flow
├── observability/   # OTEL instrumentation
├── oidc/            # OIDC discovery + endpoints
├── partnerconsent/  # Partner data sharing consent
├── passkey/         # WebAuthn/FIDO2 passkey support
├── registration/    # User registration flow
├── session/         # Redis-backed session management
├── sso/             # Single sign-on
└── testutil/        # Test helpers
```

## Architecture Notes

- KrakenD validates JWTs issued by mellon on every request
- Sessions stored in Redis with device binding
- Integrates with user-service for user lookup and security settings
- Audit events published to Kafka (structured auth audit trail)
- krakend-revoke-server handles token revocation propagation to KrakenD
- revoke-relay assists with session invalidation across clusters
- OAuth2 app credentials cached in-memory with TTL
