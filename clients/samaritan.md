# Samaritan (Mobile App)

> Paribu mobile client for iOS and Android.

- **Repo:** pikachu-exchange/samaritan
- **Stack:** React Native, Expo/EAS, TypeScript, Yarn
- **BFF:** bff-client (:8080) via KrakenD — public host `mobile.paribu.services`
- **Real-time:** ws-hub (WebSocket)
- **Last verified 2026-08-17.**

## Key Features

Trading, portfolio, deposit/withdraw (crypto+fiat), **DEX/DeFi**, BIST & US equities, charts, price
alarms, staking, KYC, passkey auth, device trust, push notifications + Live Activities, NFC passport
reader, in-app notification inbox

## Notes for agents

- The legacy DEX-PnL path (`/v1/onchain/accounts/pnl*` via bff-client) is **gone** — removed with the
  DEX CQRS cutover (onchain#241, prod 2026-08-01). Portfolio/DEX values now come from pnl-agg,
  `defi-query` and `portfolio-query`.
- The **"Bir şeyler ters gitti"** popup is bff-client's generic-error dialog (gRPC `Unavailable` → 503),
  **not** an app crash. Client-side errors live in **Sentry**, not in the backend telemetry.
- Portfolio `/v2` wiring (M4) is done at the edge; what remains is app release + old-version drain,
  so both response shapes must keep working until the drain completes.
- Testing hook-level refetch: asserting on `result.current` after a background refetch reads **stale**
  state — assert via `queryClient.getQueryState` instead.

## Related Packages (p-utilities)

- **storybook** — shared component library
- **chartPro** / **paribu-chart** — canvas-based financial chart engine
- **p-kyc** / **nitro-kyc** — on-device identity verification SDKs (backend: `estel`)
- **expo-dtt** — device trust token native module
- **passkeys** — passkey SDK
- **p-nse** — iOS Notification Service Extension (rich/mutable push)
- **react-native-nfc-passport-reader** — NFC passport reading
