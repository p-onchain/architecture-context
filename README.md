# architecture-context

Architecture context pack for AI coding agents working on BlackSwan (Paribu v6).

## Problem

AI agents (Claude Code, Codex, etc.) only see the repo they're working on. This pack gives them the broader picture — what talks to what, where to find things, and how to write code that fits the system.

## Principle

**Tell the agent WHERE to look, not WHAT it will find.** Service docs point to repos and communication patterns. The agent clones the repo and reads actual code for implementation details.

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
├── INDEX.md              ← Service map, request flow, Kafka topics, infra overview
├── conventions.md        ← Coding standards, naming, shared patterns (corekit, proto-hub)
├── services/             ← What each service does and who it talks to
├── flows/                ← Cross-service flows (trading, deposit/withdraw, auth)
└── clients/              ← Client apps (mobile, web, trading API)
```

## Keeping Fresh

- Weekly cron job scans repos and opens PRs for changes
- Team members can open PRs for their domain
