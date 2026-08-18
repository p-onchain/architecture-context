# Web Client — micro-web

> The Paribu web platform. **A single-spa + Vite + Nuxt-SSR-fragment micro-frontend monorepo**, not
> the old monolithic SPA. Same codebase is the base for a planned Tauri v2 desktop app.

- **Repo:** p-blackswan/**micro-web** (Vue; `apps/` + `packages/`; docs in its `CLAUDE.md`)
- **BFF:** bff-api (:3000) via KrakenD
- **Real-time:** ws-hub (WebSocket)
- **Serves:** `paribu.com` / `www.paribu.com` — Envoy `external-gw` routes the apex straight to
  `paribu-orchestrator:8080` (`/blog` → support-external, `/hub` → hub-external).
  `web.paribu.com` / `api.paribu.com` / `app.paribu.com` go to KrakenD instead.
- **Where it runs:** namespace **`momentum`** on exc-prod-alpha — 12 ArgoCD apps, one per
  `.cd/helm/prod/paribu-*.yaml`
- **Last verified 2026-08-17.**

⚠️ **`p-blackswan/web` (the Vue SPA) is no longer the prod frontend.** It still has a
`.cd/helm/prod/values.yaml` but no running ArgoCD app; `web-archived` also exists. Don't start work
there — check `micro-web` first.

## Architecture

```
Browser → orchestrator (single-spa registry shell)
   ├─ inline SSR fragment: header, footer
   ├─ mount: auth            on /auth/*
   ├─ mount: wallet          on /wallet
   ├─ mount: account         on /account/*
   ├─ mount: overlay         on ?overlay=<slug>
   ├─ mount: market-detail   on /markets/:asset
   ├─ mount: home            on /
   └─ proxy: LSSR            for SEO pages (/markets, /fiyat, …)
```

Deployed apps: `paribu-orchestrator`, `paribu-header`, `paribu-footer`, `paribu-home`,
`paribu-home-ssr`, `paribu-lssr` (10 pods — the SEO tier), `paribu-market-detail`,
`paribu-terminal`, `paribu-wallet`, `paribu-account`, `paribu-auth`, `paribu-overlay`.

**One Vue / one Pinia / one i18n:** the orchestrator publishes
`__PARIBU_RUNTIME__ = { Vue, Pinia, VueI18n, i18n }` at boot and every micro-app consumes it —
so a second copy of any of these is a bug, not a choice. Shared code lives in
`packages/{runtime,contracts,ui,shared,codec,formatters,build}`; the framework arrives via git
submodule (`git submodule update --init`).

## Related

- **lssr** (p-blackswan/lssr) — the standalone lightweight-SSR service the SEO tier descends from
- **storybook** (p-utilities) — shared component library
- **chartPro** / **paribu-chart** (p-utilities) — canvas financial chart engine (Nuxt 3: indicators,
  drawings, orders, replay)
- **figsmith** (p-utilities) — Figma→code pipeline targeting this stack
