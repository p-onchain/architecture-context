# Notification (notif2 + read-mono `notify`)

> Multi-channel delivery: push (FCM/APNs), SMS, e-mail, in-app inbox. Two generations coexist —
> **notif2** is the live one; `notification` (v1) is being retired.

**Last verified 2026-08-17.**

## Deployments

| App | Repo | Where |
|---|---|---|
| **`notify-api`** | p-blackswan/**notif2** | ns `saul`, exc-prod-alpha (5 pods). HTTP :8080, `notify-api.int.paribu.com` |
| `notify-iys-worker`, `notify-iys-worker-pass` | notif2 | İYS (consent registry) workers |
| `notify-projection`, `notify-query` | **read-mono** `notify` domain | ns `saul`, **exc-prod-hw only** (`notify-query.internal.paribu.com:50051`) |
| `notification-api`, `notification-worker` | p-blackswan/notification (v1) | ns `saul` — legacy |

⚠️ **Naming trap:** notif2 deploys as **`notify-api`**. The rollout named `notification-*` is the
**old v1 service**. Config/DB hunts that grep for "notification" land in the wrong service.

- **DB:** Postgres RDS `exc-prod-notification`, database **`notify`** (user `notify_appuser`;
  v1 uses `notify_service`). Device registry table `devices` (`user_id`, `token` = FCM registration
  token, `platform`, `token_status`, `notification_permission`, `deleted_at`).
  ⚠️ One user can hold **several `active` iOS rows** — `order by updated_at desc limit 1` is the only
  safe pick, and `last_seen_at` can be months stale while the row still says `active`.

## Kafka (MSK)

| Topic | Direction |
|---|---|
| `notification.request` | **ingress** — the typed contract every caller uses (`corekit/notify`, proto `notification/request/v1`) |
| `notification.request.bulk` | bulk/campaign ingress (CRM) |
| `notification.news.push` | news push |
| `user-events` | consumed — incl. the Live Activity KYC trigger |
| `transaction-events` | consumed — Live Activity transfer progress |

## Callers

`corekit/notify` is the shared typed client. ⚠️ **Retry semantics differ per caller and a notify 503
is never safe back-pressure:** staking and transaction are **fire-and-forget** (the notification is
lost), while user-service and conditional-order retry. CRM drives bulk campaigns
(`crm-*` in ns `gopanel`).

## Reading path

Web **and** mobile inbox both go through **bff-client** `/v1/notification/*` → `notify-query`
(E1B-83; the web repoint shipped in bff-client v1.3.13 on 2026-08-17, +33–41 ms measured).
`bff-api`/`bff-client` also hold `NOTIFICATION_API_URL` → `notify-api:8080` for the command side.

## Traps that have cost real time

- **Pre-outbox silent loss:** a template-lookup error is swallowed by `HasTemplate`, fan-out
  continues, and Kafka commits — **no DLQ row, no metric**. A clean DLQ does not mean no harm.
- **Template stack:** PG error → embed-fallback → DLQ is a silent path;
  `/promotional_template` CRUD is unauthenticated. Templates are single-source since EP-99;
  `notification-templates` (the repo) is archived.
- **gopanel mirrors notif2 DTOs by hand** — mirror structs silently drop new notif2 fields.
  Diff the mirror against the notif2 DTO before debugging "missing data".
- **`request_key` dedups ACTIVE batches only** (migration 043) ⇒ broadcasts need their own ledger.
- **"Success" ≠ delivered.** `send_total{success}` means the provider *accepted*. DLR outcomes have
  been metriced only since 2026-07-27; before that, non-delivery needs the history tables.
- **SMS:** Infobip credentials 401 ⇒ Mobildev is a single point of failure and foreign-number OTP is
  broken by design. The one honest OTP signal is `auth_mfa_verify_total`. Root cause of the July
  silent outages was **late delivery**, not rejection (healthy p50 ≈ 2.9 s; 1,617 messages > 5 min
  during the incident).
- **iOS stale toasts** come from notif2's blanket `content-available`; `mutable-content` is load-bearing.
- **Counter-style projections need an event-identity guard** — Live Activity saw 25× redelivery.
- **Capacity:** the notify RDS hit its gp3 IOPS ceiling under CRM bulk load (fixed 2026-08-14:
  400 GiB / 12k IOPS, r8g.large), but the sustained limit is still ~3,600 — app-side backlog
  (DB governor, index diet, UUIDv7 keys) is still ours to fix.
- `notification_event` is ~49 M rows with **no retention**.
- notif2's `main` has **no required status checks**; a skipped job reports SKIPPED, not failed.

## Related

- `flows/auth-flow.md` (OTP/MFA), `services/read-mono.md` (the `notify` read domain)
- Alerting: `platform-gitops/monitoring/alerts/pod-<team>/` — several notify alert thresholds were
  unreachable as written; check the rule, not just the dashboard
