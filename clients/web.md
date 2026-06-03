# Web Client

> Paribu web trading platform — Vue.js single-page application.

## Quick Facts

- **Repo:** p-blackswan/web
- **Stack:** Vue.js
- **BFF:** bff-api (:3001) via KrakenD
- **Real-time:** ws-hub (WebSocket)
- **SEO:** lssr (lightweight SSR framework for multi-domain SEO pages)

## Architecture

```
Browser
  → KrakenD Gateway (JWT validation)
    → bff-api (HTTP :3001)
      → gRPC microservices

  → ws-hub (WebSocket)
    → Real-time market data, order updates
```

## Related

- **micro-web** (p-blackswan) — micro frontend modules
- **support-web** (p-blackswan) — support/help center (Vue)
- **lssr** (p-blackswan) — lightweight SSR for SEO landing pages
- **prb-desktop** (p-utilities) — desktop app (Vue + Electron/Tauri)
- **storybook** (p-utilities) — shared component library
- **chartPro** (p-utilities) — financial chart engine
- **paribu-chart** (p-utilities) — chart component
