# bff-client

> Backend-for-Frontend for the mobile app (Samaritan). Same pattern as bff-api but with response shapes optimized for mobile.

- **Repo:** p-blackswan/bff-client
- **Lang:** Go
- **Port:** HTTP :8080
- **Stateless proxy** — no DB

## Talks To

Same downstream services as bff-api (see `bff-api.md`), with mobile-specific response mapping.

## Key Details

- Sits behind KrakenD
- No Kafka
- Uses corekit + proto-hub
- Feature flags via flagd/OpenFeature

## Notable Endpoints & Behaviors (2026-05/06)

- **GET /v1/banners** — hardcoded home banners response (no downstream call); response shapes owned in `internal/feature/banner/` (EXCH-5185, 2026-06)
- **GET /v1/onchain/health** — public health probe; IP rate-limited at gateway (60 req/min)
- **POST /v1/onchain/…/challenge** — X-Forwarded-For forwarded by gateway for audit (fixed 2026-06)
- **Helpdesk attachment upload** — body limit raised 10 MiB → **25 MiB**, timeout 30s → 60s (EXCH-6915); bff-client's `http.MaxBytesReader` is the authoritative cap; gateway rejects `Transfer-Encoding: chunked` (no `Content-Length`) before the body reaches bff-client; ClamAV scan limit is 30M — raises above 25 MiB require coordinated ClamAV config change

## Recent API Additions (EXCH-5185, 2026-06)

- **Portfolio:** staking APY returned per portfolio item; loyalty balance aligned with portfolio total
- **Alarm history:** `type` and `caip19` (full CAIP-19 identifier) fields added to alarm history response
- **Feedback:** returns `has_feedback: bool` instead of 404 when feedback is absent
- **Markets / asset list:** column sort keys added; open-orders now filter by `?market=` query param
- **KYC video upload:** `SERVER_READ/WRITE_TIMEOUT` bumped 15s → 120s; graceful shutdown drain added to avoid mid-upload kills (EXCH-5185)
