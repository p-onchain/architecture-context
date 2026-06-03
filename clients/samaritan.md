# Samaritan (Mobile App)

> Paribu mobile client — React Native (Expo) app for iOS and Android.

## Quick Facts

- **Repo:** pikachu-exchange/samaritan
- **Stack:** React Native, Expo, TypeScript
- **BFF:** bff-client (:8080) via KrakenD
- **Real-time:** ws-hub (WebSocket for market data, order updates)

## Architecture

```
Samaritan (Expo)
  → KrakenD Gateway (JWT validation)
    → bff-client (HTTP :8080)
      → gRPC microservices

  → ws-hub (WebSocket)
    → Real-time market data, order updates, notifications
```

## Key Features

- Trading (spot orders, conditional orders, recurring buys)
- Portfolio & balance management
- Deposit/withdrawal (crypto + fiat)
- Market data (charts via chartPro, orderbook, ticker)
- Price alarms
- Staking
- KYC (via paribu-kyc SDK)
- Passkey authentication
- Device trust (expo-dtt native module)
- Push notifications
- NFC passport reader (KYC)
- Stock trading surfaces (BIST/US via xpr-stock)

## Related Packages (p-utilities)

- **storybook** — shared component library
- **chartPro** — canvas-based financial chart engine (Nuxt 3)
- **paribu-kyc** — on-device identity verification SDK
- **expo-dtt** — device trust token native module
- **passkeys** — passkey SDK
- **preset-paribu** — shared presets/config
