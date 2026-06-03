# Flow: Authentication & Authorization

> How users authenticate and how requests are authorized across the platform.

## Login Flow

```
1. Client (Samaritan/Web) → KrakenD → mellon
   → POST /auth/login (email + password)
   → mellon validates credentials against user-service
   → If MFA enabled:
     → Returns MFA challenge (TOTP / passkey / SMS)
     → Client submits MFA response
     → mellon verifies

2. Token issuance:
   → mellon issues JWT (access token + refresh token)
   → Session stored in Redis
   → JWT contains: user_id, session_id, roles, expiry

3. Client stores tokens, uses access token for subsequent requests
```

## Request Authorization

```
Every API request:
  Client → KrakenD
    → KrakenD validates JWT signature (mellon's public key)
    → KrakenD checks token expiry
    → KrakenD extracts user_id, injects as header (X-User-Id)
    → Routes to BFF (bff-api or bff-client)
    → BFF passes user_id to downstream gRPC services via metadata
```

## Token Refresh

```
Client → KrakenD → mellon
  → POST /auth/refresh (refresh token)
  → mellon validates refresh token + session
  → Issues new access token
  → Returns to client
```

## Logout / Session Revocation

```
Client → KrakenD → mellon
  → POST /auth/logout
  → mellon invalidates session in Redis
  → krakend-revoke-server propagates revocation to all KrakenD instances
  → revoke-relay handles cross-cluster propagation
  → Produces: all_sessions_closed_event (if logout-all)
```

## API-Key Authentication (Trading API)

```
Trading client → KrakenD
  → krakend-apikey-plugin validates:
    1. HMAC signature (timestamp + method + path + body)
    2. API key lookup
    3. IP whitelist check
    4. Rate limit check (tiered, weight-based)
  → Injects X-User-Id header
  → Routes to wapi (WebSocket) or bff-api (REST)
```

## Passkey (WebAuthn) Flow

```
Registration:
  Client → bff → user-service.PasskeyService
  → WebAuthn challenge → client signs with device → verify + store

Login:
  Client → mellon → WebAuthn assertion
  → mellon verifies against stored passkey
  → Issues JWT (same as password login)
```

## Device Trust Token (DTT)

```
Mobile app uses DTT (native module: expo-dtt):
  → Device generates trust token on first login
  → Token bound to device + user
  → Subsequent logins validated with DTT
  → Reduces MFA friction for trusted devices
```

## Key Components

| Component | Role |
|-----------|------|
| **mellon** | Auth service (OIDC, sessions, JWT issuance) |
| **KrakenD** | JWT validation on every request |
| **krakend-apikey-plugin** | API-key HMAC auth for trading clients |
| **krakend-revoke-server** | Token revocation propagation |
| **revoke-relay** | Cross-cluster session invalidation |
| **user-service** | User credentials, MFA settings, passkeys |
| **expo-dtt** | Device trust tokens (mobile native) |
