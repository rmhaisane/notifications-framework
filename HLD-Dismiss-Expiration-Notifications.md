# Dismiss Expiration Notifications by User — HLD

| | |
|---|---|
| **Feature / Epic** | [PTCI-4043 — Customer can ignore/dismiss Expiration Notifications by User](https://infoblox.atlassian.net/browse/PTCI-4043) |
| **Target release** | 26.4 |
| **Author** | Notifications team |
| **Status** | Draft |
| **Related HLDs** | [Atlas Notifications Infrastructure HLD](https://infoblox.sharepoint.com/:w:/s/Engineering/IQDppco2_-H5RYqFq7Q-VM7WAbdD8M-Y9UXaIRqBDlqPq3I) · [Notifications Optimize In-App Sender and Mailbox](https://infoblox.sharepoint.com/sites/Engineering/_layouts/15/Doc.aspx?sourcedoc=%7BC22A8E33-4864-489F-A159-350BB5DF0A70%7D&file=Notifications%20Optimize%20In-App%20Sender%20and%20Mailbox.docx) · [Notifications Survey Generation HLD](https://infoblox.sharepoint.com/sites/Engineering/_layouts/15/Doc.aspx?sourcedoc=%7BC37829E8-ED51-4F4A-B236-7C0538A932F1%7D&file=Notifications%20Survey%20Generation%20HLD.docx) |

---

## 1. Overview

### 1.1 Purpose

Allow Infoblox portal users to **permanently dismiss** expiring-subscription notifications on a **per-user, per-subscription** basis, from both the **portal** and the **email** channel, while ensuring the user makes an informed decision before suppressing future warnings.

### 1.2 Problem statement

Users today receive recurring in-app and email warnings about expiring subscriptions even after they have made (or deliberately deferred) a renewal decision. The lack of a dismiss mechanism produces alert fatigue, clutters the mailbox, and erodes the value of all notifications across the platform.

### 1.3 Objective

Deliver a per-user dismiss capability that:

- Suppresses every future expiring-subscription warning for the targeted `(user, subscription)` pair on every channel.
- Cannot be triggered accidentally (explicit confirmation in both channels).
- Cannot be tampered with from email (signed, time-limited, single-use token).
- Is fully auditable and operationally observable.

### 1.4 Out of scope

- Dismissing notification *types* other than expiring-subscription warnings.
- Account-level dismissal (the FS is strictly per-user — FR-9).
- Reversal from the customer-facing UI (FR-2 specifies it cannot be undone). An internal support override is provided.
- Recall of emails already in the user's inbox.

---

## 2. Requirements traceability

| Requirement | Met by |
|---|---|
| FR-1 Dismiss action in portal | UI Dismiss button on every expiring-subscription mailbox item |
| FR-2 Confirmation prompt | Modal with explicit "Yes, permanently dismiss" / Cancel |
| FR-3 Suppression scope | `subscription_dismissals` row + processor/dispatcher suppression |
| FR-4 Immediate removal | Mailbox archives matching items inside the same transaction |
| FR-5 Dismiss link in email | Postoffice template extension with `{{ .DismissURL }}` |
| FR-6 Authenticated dismissal | Signed JWT + portal authenticated confirmation page |
| FR-7 Confirmation page | Postoffice-hosted page → user clicks Confirm |
| FR-8 Suppression parity | Both channels call the same internal Dismiss API |
| FR-9 Per-user scope | Composite PK `(user_id, subscription_id)` |
| FR-10 Persistence | Server-side Postgres row in mailbox DB |
| NFR-1 Security | HMAC-signed JWT, `exp ≤ 7d`, single-use via Redis `jti` |
| NFR-2 Performance | Single indexed write; cached read on send path; SLO < 2s |
| NFR-3 Auditability | Audit-log emission on every dismissal write |

---

## 3. System architecture

### 3.1 Affected services

| Service | Change |
|---|---|
| [atlas.notifications.mailbox](../../atlas.notifications.mailbox) | New `subscription_dismissals` table, Dismiss APIs (user + S2S), `dismissal.changed` Kafka event, archive on dismiss, audit log |
| [atlas.notifications.postoffice](../../atlas.notifications.postoffice) | JWT signer/verifier, `/v1/dismiss/confirm` page, S2S call to mailbox |
| [atlas.notifications.config](../../atlas.notifications.config) | Template extension (`dismissible`, `{{ .DismissURL }}`), default-template migration |
| [atlas.notifications.dispatcher](../../atlas.notifications.dispatcher) | New dismissal cache + Kafka handler; recipient filter on send path |
| [atlas.notifications.events.processor](../../atlas.notifications.events.processor) | Optional source-side suppression for expiring-subscription events |
| [atlas.notifications.sender](../../atlas.notifications.sender) | Pass-through; no behavioral change in 26.4 |
| [athena.license.service](../../athena.license.service) | (Future) consume `dismissal.changed` to skip event generation |
| CSP portal frontend | Dismiss button, confirmation modal, email confirmation landing page |

### 3.2 High-level flow

```
                ┌────────────────────┐
  License svc → │ events.processor   │ ── checks dismissal cache (skip)
                └─────────┬──────────┘
                          │ Kafka  notifications.events
                          ▼
                ┌────────────────────┐
                │ dispatcher         │ ── drops dismissed users from recipients
                └────┬───────────┬───┘
              in-app│           │email
                    ▼           ▼
              ┌──────────┐ ┌──────────────┐
              │ mailbox  │ │ postoffice   │── injects signed Dismiss URL
              └────┬─────┘ └──────┬───────┘
   POST /dismiss  │              │  GET /v1/dismiss/confirm?token=JWT
   (user-auth)    │              │  → confirmation page → Confirm
                  ▼              ▼
            ┌──────────────────────────┐
            │ mailbox: Dismiss service │
            │  - writes subscription_  │
            │    dismissals row        │
            │  - archives current items│
            │  - emits Kafka           │
            │    dismissal.changed     │
            │  - audit log             │
            └──────────────────────────┘
                          │
                          ▼ Kafka  notifications.dismissals.changed
              cache invalidation in
              processor + dispatcher
```

---

## 4. Data model

### 4.1 New table — `subscription_dismissals` (mailbox DB)

```sql
CREATE TABLE subscription_dismissals (
    user_id          TEXT        NOT NULL,
    subscription_id  TEXT        NOT NULL,
    account_id       TEXT        NOT NULL,
    scope            TEXT        NOT NULL DEFAULT 'subscription',
    channel          TEXT        NOT NULL,                  -- 'portal' | 'email'
    dismissed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, subscription_id)
);

CREATE INDEX ix_dismissals_account ON subscription_dismissals (account_id);
CREATE INDEX ix_dismissals_subscription ON subscription_dismissals (subscription_id);
```

- `scope` is reserved for future expansion (e.g. `type`, `category`) — keeps the API stable.
- Cascade delete is handled via account/user lifecycle hooks (existing pattern in mailbox).

### 4.2 Existing per-user notification status

The per-user notification-status table introduced in *Notifications Optimize In-App Sender and Mailbox* gains a `dismissed` flag. `GET /v1/mailbox/notifications` filters out rows where either the per-row `dismissed=true` or a matching `subscription_dismissals` row exists.

### 4.3 Single-use token store (Redis, postoffice)

`dismiss:jti:<jti>` → `1`, `EXPIREAT = token.exp`. Atomic `SET NX` enforces single-use.

---

## 5. APIs

### 5.1 Mailbox — user-facing

```
POST /v1/mailbox/dismissals
Authorization: Bearer <user JWT>
Content-Type: application/json
{
  "subscription_id": "sub_abc123"
}

201 Created
{
  "user_id": "...", "subscription_id": "sub_abc123",
  "account_id": "...", "dismissed_at": "...", "channel": "portal"
}
```

```
GET  /v1/mailbox/dismissals
DELETE /v1/mailbox/dismissals/{subscription_id}     # not exposed in UI; admin-only
```

### 5.2 Mailbox — service-to-service

```
POST /v1/internal/mailbox/dismissals
Authorization: Bearer <S2S JWT>
{
  "user_id": "...",
  "subscription_id": "...",
  "account_id": "...",
  "channel": "email"
}
```

Used by postoffice after token verification; uses Athena S2S claims (existing pattern).

### 5.3 Postoffice — email confirmation

```
GET  /v1/dismiss/confirm?token=<JWT>
        → renders confirmation page (FR-7)
POST /v1/dismiss/confirm
        → verifies token, calls mailbox S2S endpoint, renders success page
```

### 5.4 Token specification (NFR-1)

```jsonc
{
  "iss": "notifications-postoffice",
  "aud": "notifications-postoffice",
  "purpose": "dismiss_expiry",
  "user_id":         "...",
  "account_id":      "...",
  "subscription_id": "...",
  "jti":  "<uuid v4>",
  "iat":  <unix>,
  "exp":  <iat + 7d>          // configurable
}
```

- Algorithm: **HS256** with key from existing `pkg/secrets`, rotated.
- Single-use enforced by `jti` in Redis; second redemption returns `410 Gone`.
- `purpose` claim prevents the same token shape being reused for any other intent.

---

## 6. Workflows

### 6.1 In-portal dismissal (US-1, FR-1..FR-4)

1. User opens mailbox → sees expiring-subscription card with **Dismiss** button.
2. Click → modal: *"This permanently stops all future warnings for this subscription. This cannot be undone."* — **Confirm / Cancel**.
3. UI → `POST /v1/mailbox/dismissals`.
4. Mailbox in one transaction:
   - inserts/`ON CONFLICT DO NOTHING` into `subscription_dismissals`,
   - sets `dismissed=true` on every existing in-app row for `(user_id, subscription_id)`,
   - emits `dismissal.changed` Kafka event,
   - emits audit-log line.
5. Response `201` → UI removes the card immediately. SLO < 2s (NFR-2).

### 6.2 Email dismissal (US-2, FR-5..FR-8, NFR-1)

1. Postoffice renders the expiring-subscription email; template includes
   `https://csp.<env>.infoblox.com/notifications/dismiss?token=<JWT>`.
2. User clicks → if not authenticated, Athena login → `GET /v1/dismiss/confirm?token=…`.
3. Confirmation page shows: subscription name, irreversibility warning, **Confirm** / **Cancel**.
4. **Confirm** → `POST /v1/dismiss/confirm`:
   - verify signature, `purpose`, `exp`,
   - `SET NX dismiss:jti:<jti>` — fail if already used,
   - call mailbox `POST /v1/internal/mailbox/dismissals` with `channel="email"`,
   - render success page.
5. Mailbox path identical to §6.1 (FR-8 parity).

### 6.3 Suppression at the source

Events processor materializes recipients for an expiring-subscription event, then:

```go
recipients = recipients.Filter(func(uid string) bool {
    return !dismissalsCache.IsDismissed(uid, event.SubscriptionID)
})
if len(recipients) == 0 { return nil } // skip publish
```

### 6.4 Suppression at fan-out (safety net)

Dispatcher applies the same filter when expanding the recipient list, after subscription/filter resolution. This protects against events generated by future producers that bypass the processor check.

### 6.5 Cache invalidation

Mailbox publishes to `notifications.dismissals.changed`:

```jsonc
{
  "event": "dismissed" | "undismissed",
  "user_id": "...",
  "subscription_id": "...",
  "account_id": "...",
  "ts": "..."
}
```

Dispatcher (`pkg/mq/handler_dismissals_changes.go`) and processor consume this and update their in-memory caches (TTL fallback 5 min).

---

## 7. Security

| Concern | Mitigation |
|---|---|
| Forged email link | HMAC-signed JWT; rotated key |
| Replay | `jti` single-use in Redis; TTL = token `exp` |
| Token theft from inbox | `exp ≤ 7d` (configurable, NFR-1); user must still confirm on the portal |
| CSRF on confirmation page | Standard Athena CSRF tokens; `POST` required for the actual write |
| Cross-tenant dismissal | Token `account_id` claim must match the authenticated user's account; mismatch → `403` |
| Unauthorized escalation | Internal endpoint is S2S only; `purpose` claim namespaces tokens |
| Audit | Every write produces an audit-log entry (NFR-3) |

OWASP coverage: A01 (access control via Athena + claim check), A02 (signed tokens), A04 (single-use, expiry, purpose binding), A09 (audit logging).

---

## 8. Observability

**Metrics**

- `dismissals_created_total{channel="portal"|"email"}`
- `dismissals_undone_total{actor="admin"}`
- `notifications_suppressed_total{reason="dismissed", layer="processor"|"dispatcher"}`
- `dismiss_token_invalid_total{reason="signature"|"expired"|"reused"|"purpose"}`
- `dismiss_api_duration_seconds` (histogram, SLO < 2s)
- `dismissals_cache_hit_ratio`

**Logs** — structured JSON, includes `user_id`, `account_id`, `subscription_id`, `channel`, `jti` (where applicable).

**Audit log** — pushed to existing Athena audit pipeline.

**Alerts**

- `dismiss_token_invalid_total{reason="reused"}` rate spike → potential abuse.
- `dismiss_api_duration_seconds:p99 > 2s` → NFR-2 violation.
- Kafka consumer lag on `notifications.dismissals.changed` > 30s.

---

## 9. Migration & rollout

### 9.1 DB migration

Sequenced migration in [atlas.notifications.mailbox/db/migrations](../../atlas.notifications.mailbox/db/migrations) — `XXXX_subscription_dismissals.up.sql` / `.down.sql`.

### 9.2 Template migration

In [atlas.notifications.config/db/migrations](../../atlas.notifications.config/db/migrations): an `update_default_templates` migration adds the `{{ .DismissURL }}` block and `dismissible: true` flag for the expiring-subscription template only.

### 9.3 Feature flag

`notifications.expiry_dismiss.enabled` (CRD-driven, mirrors *Optimize In-App* HLD pattern):

| Phase | Flag | Behavior |
|---|---|---|
| 1 | `false` | DB + APIs deployed; UI hidden; emails unchanged |
| 2 | `true` (in-app only) | Portal Dismiss visible; suppression active |
| 3 | `true` (full) | Email link injected; full E2E |

### 9.4 Rollback

- Toggle flag `false` → UI hides Dismiss; suppression filter becomes a no-op; existing rows remain (no data loss).
- DB migration is reversible; emails revert via template version pin.

---

## 10. Performance considerations

- Dismiss write: 1 indexed insert + 1 update on per-user-status rows for that subscription (typically < 10 rows).
- Suppression read: cache hit (in-process map keyed by `account_id`); cold path is a single indexed lookup.
- Email send path: no extra DB call — token is signed in memory.
- Expected dismissal volume is low (single-digit per user per year); not a hot path.

---

## 11. Dependencies

- Athena auth (user + S2S Bearer tokens).
- Existing notifications Kafka pubsub (Dapr).
- Existing Redis used by postoffice.
- `pkg/secrets` for JWT key material and rotation.
- Per-user notification-status table from *Optimize In-App Sender/Mailbox* HLD.

---

## 12. Story breakdown

### Backend — mailbox
1. Migration: `subscription_dismissals` table + indexes.
2. Per-user-status `dismissed` flag + read-path filter.
3. `POST /v1/mailbox/dismissals` (user) + archive logic.
4. `POST /v1/internal/mailbox/dismissals` (S2S).
5. `GET /v1/mailbox/dismissals` listing.
6. Admin `DELETE` override endpoint.
7. Publish `dismissal.changed` Kafka event.
8. Audit-log emission.

### Backend — suppression
9. Dispatcher: `pkg/mq/handler_dismissals_changes.go` + cache + recipient filter.
10. Events processor: source-side suppression + cache.

### Backend — email
11. Config: template schema flag `dismissible`, placeholder `{{ .DismissURL }}`, default-template migration.
12. Postoffice: JWT signer/verifier wired to `pkg/secrets`.
13. Postoffice: `GET/POST /v1/dismiss/confirm` + confirmation page.
14. Postoffice: single-use `jti` Redis enforcement.

### License integration (optional for 26.4)
15. License service consumes `dismissal.changed` (or queries mailbox) to skip generating events.

### UI (CSP portal — separate repo)
16. Dismiss button on expiring-subscription card.
17. Confirmation modal (FR-2 wording).
18. Email-link confirmation landing page (FR-7).
19. Optional "View dismissed subscriptions" page.

### Cross-cutting
20. Metrics + dashboards.
21. Security review (token format, key rotation, replay).
22. Performance test (NFR-2).
23. Audit pipeline verification (NFR-3).
24. Runbook updates (`Notifications Runbook - Debugging`).
25. Feature flag `notifications.expiry_dismiss.enabled`.

---

## 13. Open questions

- **Already-delivered emails**: confirm UI copy clearly says "future warnings only".
- **Account hierarchies / MSPs**: per FR-9 strictly per-user; confirm with PM for managed accounts.
- **Token TTL default**: 7 days proposed (NFR-1 max); confirm with security.
- **Where does the source-side suppression live for licenses generated upstream?** Decision needed between events.processor and license-service.
- **Reversal policy**: keep admin-only override, or also expose a "View dismissed" page where the user can re-enable warnings?

---

## 14. Appendix — affected source paths (initial estimate)

- [atlas.notifications.mailbox/db/migrations](../../atlas.notifications.mailbox/db/migrations)
- [atlas.notifications.mailbox/pkg/svc](../../atlas.notifications.mailbox/pkg/svc)
- [atlas.notifications.mailbox/channels](../../atlas.notifications.mailbox/channels)
- [atlas.notifications.dispatcher/pkg/mq](../../atlas.notifications.dispatcher/pkg/mq)
- [atlas.notifications.dispatcher/pkg/cache](../../atlas.notifications.dispatcher/pkg/cache)
- [atlas.notifications.events.processor/pkg/event](../../atlas.notifications.events.processor/pkg/event)
- [atlas.notifications.config/db/migrations](../../atlas.notifications.config/db/migrations)
- [atlas.notifications.postoffice/pkg/email](../../atlas.notifications.postoffice/pkg/email)
- [atlas.notifications.postoffice/pkg/svc](../../atlas.notifications.postoffice/pkg/svc)
