# architecture-context

Architecture context pack for AI coding agents working on BlackSwan (Paribu v6).

- **Repo:** p-onchain/architecture-context

## Problem

AI agents (Claude Code, Codex, etc.) only see the repo they're working on. This pack gives them the
broader picture — what talks to what, where to find things, and how to write code that fits the system.

## Principle

**Tell the agent WHERE to look, not WHAT it will find.** Service docs point to repos and communication
patterns. The agent clones the repo and reads actual code for implementation details.

The one exception: **facts an agent cannot derive from a single repo** — which cluster/cloud a service
runs in, which of the two Kafka clusters it uses, which CI checks are no-ops, which repos are archived.
Those are recorded explicitly, with the date they were verified.

## Usage

Add to `~/.claude/CLAUDE.md`:
```
For BlackSwan architecture context, see ~/architecture-context/INDEX.md
```

Or per-repo in `CLAUDE.md`:
```
For broader architecture context, see ~/architecture-context/INDEX.md
```

## Structure

```
├── INDEX.md              ← Clusters/namespaces, request flow, Kafka topology, topics, infra, orgs
├── conventions.md        ← Coding standards, ticket/branch rules, the deploy mechanism, CI traps
├── services/             ← What each service does and who it talks to
│   ├── read-mono.md          (the CQRS read side — 28 domains)
│   ├── onchain.md            (DeFi / Wallet-as-a-Service)
│   ├── notification.md       (notif2 + read-mono notify)
│   ├── gateway.md            (Cloudflare → Envoy Gateway → KrakenD)
│   └── …                     (order-api, match, wallet, transaction, user-service, mellon, bff-*, supporting)
├── flows/                ← Cross-service flows (order lifecycle, deposit/withdraw, auth)
├── clients/              ← Client apps (micro-web, Samaritan, WAPI)
└── test-env.md           ← Reaching the live test cluster: VPN/SSO, kubectl, in-pod DB & vault access
```

## Keeping Fresh

Each doc carries a **"Last verified"** date. Trust the date, not the prose.

There is no working automated refresh right now — the pack went ~2 months stale between the
2026-06-15 and 2026-08-17 passes. Refresh it by hand when you notice drift, or ask an agent to.
A refresh pass that actually catches architectural change checks, in order:

1. **Live prod** via sre-mcp — `sre_list_services` for the ArgoCD app inventory and
   `vm_query` on `count by (cluster, namespace, created_by_name) (kube_pod_info{...})` for where pods
   *really* run (ArgoCD's reported namespace is not the pod namespace)
2. **`p-platform/platform-gitops`** — `app-of-apps/` (how apps are generated), `applications/*/values/<cluster>.yaml`
   (edge, brokers, SR, flags), `monitoring/alerts/` (who gets paged)
3. **Each repo's `.cd/helm/<env>/*.yaml`** — the ground truth for ports, namespace, brokers, topics, datastores
4. **`gh repo list <org>`** — new repos, and especially newly **archived** ones
