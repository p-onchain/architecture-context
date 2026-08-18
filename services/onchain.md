# onchain (DeFi / Wallet-as-a-Service)

> The on-chain platform: ERC-4337 smart accounts, DEX swaps, Hyperliquid perps, Polymarket, and
> crypto deposit detection. The one big **TypeScript/Bun (NestJS)** service in an otherwise Go fleet.

- **Repo:** p-blackswan/**onchain** (⚠️ `p-blackswan`, not `p-onchain`) · Bun + NestJS monorepo
  (`apps/`, `packages/`)
- **Namespace:** `onchain` on `exc-prod-alpha` (~20 pods)
- **DB:** PostgreSQL `onchain_db`
- **Last verified 2026-08-17** (onchain@origin/main + live prod pods)

## Deployed apps

| App | Repo path | Notes |
|---|---|---|
| `defi-api` | `apps/api` | main API, HTTP **:8080** (`defi-api.onchain.svc.cluster.local:8080`, `defi-api.int.paribu.com`). 8 pods |
| `defi-magic-spend` | `apps/magic-spend` | ERC-4337 paymaster / MagicSpend |
| `defi-scheduler-critical` | `apps/scheduler-critical` | hot cron + **transactional-outbox relay** |
| `defi-scheduler-periodic` | `apps/scheduler-periodic` | slow cron (incl. new-account address POSTs) |
| `defi-hyperliquid-stream` | `apps/hyperliquid-stream` | Hyperliquid market/position stream |
| `defi-polymarket-stream` | `apps/polymarket-stream` | Polymarket price/resolution stream |
| `defi-bundler-{base,bsc,polygon,hyperevm,robinhood}` | **p-blackswan/bundler-erc4337** | one ERC-4337 bundler per chain |

Shared packages: `api-core`, `dex-aggregator`, `smart-account`, `token-data`, `hyperliquid-client`,
`passkeys`, `shared`, `observer-ts`.

## Chains

`eip155:1` Ethereum · `eip155:56` BSC · `eip155:137` Polygon · `eip155:8453` Base ·
`eip155:999` HyperEVM · Solana (SVM) · **Robinhood Chain** (bundler deployed).
Adding a chain = deploy pinned contracts → add-chain → list tokens.

## Kafka (MSK)

Topic + client-id registry: `packages/api-core/src/constants/kafka.ts` — read it, don't guess.

**Produces**

| Topic | Consumer |
|---|---|
| `order.events.dex` (**v2**, `proto.dex.order.v2`) | read-mono **defi**-projection |
| `ticker.daily`, `ticker.last24h` | read-mono **pnl**-projection — onchain publishes **DeFi-token** prices onto the *same* topics read-mono's ticker-worker uses for CEX prices |
| `order.events.hyperliquid`, `hyperliquid-ledger-events`, `hyperliquid-position-events` | perps domain |
| `order.events.polymarket` | prediction-market domain |
| `defi-audit-events` | audit |
| `websocket-events`, `user-events` | ws-hub / user pipeline |

Message keys are **per-user** (`onchain-balances-<userId>`, `onchain-swap-status-<userId>`,
`onchain-polymarket-positions-<userId>`) or per-coin (`onchain-hyperliquid-assetctx-<coin>`) so a
user's/coin's updates stay ordered on one partition.

**Legacy / dormant**

- `order.events.defi` (**v1**, `proto.defi.order.v1`) — idle; retained for financial-history
- `pnl.defi.asset.updates` — ⚠️ **the onchain producer still exists in code**, drained by
  `defi-scheduler-critical`'s outbox relay (`DefiAssetUpdateOutboxHandler`), but is **flag-disabled**:
  `PUBLISH_LEGACY_DEFI_PNL_UPDATES: "false"` in both `.cd/helm/prod/` and `.cd/helm/test/`
  `defi-scheduler-critical.yaml`. The zod schema **defaults to `"true"`**, so a values file that
  drops the key silently re-enables it — and then *two* services publish the topic that read-mono
  `pnl` consumes. Post-cutover the intended sole producer is read-mono `defi`.

## Talks To

- **read-mono `defi`** — via `order.events.dex`; defi owns the DEX read model and serves
  `GetUserHoldings` back to pnl-agg
- **bff-client** — `ONCHAIN_API_URL=http://defi-api.onchain.svc.cluster.local:8080`
- **heimdall** — pulls the token catalog: `defi-api…:8080/v1/tokens/catalog`
- **transaction** — crypto deposit detection / confirmation callbacks
- **Admin panel:** `p-backoffice/paribu-one-defi` (module `gopanel-defi`) — *not* in `p-blackswan/gopanel`

## DB traps (verified in-cluster)

- `balance_positions` is keyed by **`account_id`** (bigint), not `user_id` — join `accounts`
  (`accounts.id` → `bp.account_id`; `accounts.user_id` is the UUID)
- **Token decimals live in `token_contracts.decimals`**, keyed by (`contract_address`,
  `chain_reference`). `tokens` holds only id/symbol/name/logo_url and has **no address column**.
  `balance_positions.balance` is raw base units — you need those decimals for anything human-readable.
- Bun images ship a built-in SQL client, so in-pod queries are easy — see `test-env.md`

## Related repos (p-onchain org)

`p-accounts` / `p-accounts-sol` (ERC-4337/7579 + Solana smart accounts, passkey + Guardian recovery),
`MagicSpend`, `kernel` (ZeroDev), `7579-plugins`, `swig-wallet`, `svm-account`, `legolas`
(multi-chain indexer), `token-list` (DEX listing candidates), `paribu-onchain-product` (business doc).
