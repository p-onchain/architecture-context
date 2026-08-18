# Flow: Authentication & Authorization

> How users authenticate and how requests are authorized across the platform.
> **Last verified 2026-08-17.** mellon is deployed as **`auth-service`** (ns `shelby`,
> `auth.int.paribu.com`, HTTP :8080; BFF env vars still say `MONOSIGN_*`). Public auth host is
> `https://account.paribu.com`. Requests reach KrakenD through Cloudflare → Envoy `external-gw`.

## Login Flow

```
1. Client (Samaritan/micro-web) → Envoy → KrakenD → mellon (auth-service)
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
    (bff-api reaches it at http://krakend-revoke-server.blackswan.svc.cluster.local:8081)
  → Produces: all_sessions_closed_event (if logout-all) + auth-audit-events
```

⚠️ **revoke-relay is archived** — cross-cluster propagation no longer runs through it. If you need to
reason about revocation across `exc-prod-alpha` ↔ `exc-prod-hw`, read the current
krakend-revoke-server code rather than assuming a relay exists.

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

## KYC

KYC is **not** in mellon or user-service — it lives in **`estel`** (Rust): `kyc-api`,
`kyc-orchestrator`, `kyc-backoffice`, reached by bff-client as
`PKYC_API_URL=http://kyc.internal.paribu.com` (Huawei side). Client SDKs: `p-utilities/p-kyc`,
`nitro-kyc`, `react-native-nfc-passport-reader`. user-service still owns `kyc_status` as raw state,
and there is **no test-env KYC bypass** (the real provider runs in test too).

## OTP / SMS caveats

- The only honest signal for OTP success is **`auth_mfa_verify_total`** — provider "success" only means
  the SMS was *accepted*.
- Infobip credentials are 401, so **Mobildev is a single point of failure** and foreign-number OTP is
  broken by design.
- "Parolamı unuttum": if e-mail and phone don't match, **no SMS is ever sent** while the UI still says
  a code was sent (a deliberate decoy flow) — a large share of attempts, and unalarmed.

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
| **mellon** (deployed `auth-service`) | Auth service (OIDC, sessions, JWT issuance) |
| **Envoy Gateway** | TLS/routing at `external-gw` before KrakenD |
| **KrakenD** (`krand`) | JWT validation on every request |
| **krakend-apikey-plugin** | API-key HMAC auth for trading clients |
| **krakend-revoke-server** | Token revocation propagation |
| ~~revoke-relay~~ | **archived** — no longer in the path |
| **user-service** (Huawei) | User credentials, MFA settings, passkeys |
| **estel** | KYC (`kyc-api` / `-orchestrator` / `-backoffice`) |
| **notif2** | OTP delivery (SMS/push) |
| **expo-dtt** | Device trust tokens (mobile native) |
