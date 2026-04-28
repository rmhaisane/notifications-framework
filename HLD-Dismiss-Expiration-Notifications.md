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
| FR-3 Suppression scope | One `subscription_dismissals` row → suppression on **both** in-app and email channels (§4.1.1); enforced in mailbox + dispatcher + postoffice |
| FR-4 Immediate removal | Mailbox archives matching items inside the same transaction |
| FR-5 Dismiss link in email | Postoffice template extension with `{{ .DismissURL }}` |
| FR-6 Authenticated dismissal | Signed JWT + portal authenticated confirmation page |
| FR-7 Confirmation page | Postoffice-hosted page → user clicks Confirm |
| FR-8 Suppression parity | Both channels call the same internal Dismiss API; the resulting row suppresses both `in_app` and `email` deliveries (§4.1.1) |
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

## 3.3 Subscription identity (canonical definition)

Throughout this HLD `subscription_id` means **the `License.ID` UUID owned by [athena.license.service](../../athena.license.service)** — i.e. one row in the `licenses` table:

```go
// athena.license.service/pkg/svc/licenses/model/license.go
type License struct {
    ID         uuid.UUID   // <-- this is our subscription_id
    AccountID  int64
    Sku        string
    StartDate  time.Time
    EndDate    time.Time
    ...
}
```

**Why `License.ID` and not anything else**

| Candidate | Why we did *not* pick it |
|---|---|
| `(account_id, sku)` tuple | Multiple co-existing licenses per SKU per account (renewals, evaluations, overlapping terms). A user dismissing one term must not silence a future renewal that legitimately re-enters the warning window. |
| `sku` alone | Account-wide; violates FR-9 per-user *and* per-subscription scope. |
| `entitlement_id` / `package_id` | Higher-level grouping; one entitlement can map to many `License` rows over time. Wrong granularity. |
| `fingerprint` (alertmanager) | Recomputed per event by the notifications stack from `subtype + accountID + Location`. Opaque, not stable across template changes. Bad as a public key. |

`License.ID` is the only identifier that is **stable, unique, and 1:1 with a billable term** — exactly what "this subscription" means to the user.

**It is already on every expiry event** — verified in [athena.license.service/cmd/server/expired.go](../../athena.license.service/cmd/server/expired.go):

```go
func PublishEvent(subtype, accountID, subject, message, licenseID string, status pb.Event_Status) error {
    ...
    msg := &pb.Event{
        Type:          pb.Event_ACCOUNT,
        Subtype:       subtype,           // "entitlement-expired" | "entitlement-expiry-warning"
        AccountId:     accountID,
        ApplicationId: cli.cfg.AppID,
        Location:      licenseID,         // <-- License.ID UUID, used today for dedup
        Severity:      pb.Event_high,
        Status:        status,            // RAISED | CLEARED
        ProductName:   "account",
        OccurredTime:  timestamppb.Now(),
        Metadata:      metadata,          // short_subject, message, accountID
    }
    return cli.notifClient.Publish(context.Background(), msg)
}
```

`RunExpiredHandler()` drives all three call-sites that publish expiry events (`entitlement-expired`, `entitlement-expiry-warning`, and the `CLEARED` renewal sweep). **Every one of them passes a non-empty `License.ID.String()` as `Location`**, so dispatcher and mailbox can always extract the dismissal key without any change to the upstream contract.

**Required changes for clarity (small but mandatory)**

To stop relying on the overloaded `Location` field everywhere downstream, we will:

1. Add an explicit `subscription_id` entry to `Event.Metadata` in `PublishEvent`, set to `licenseID`. `Location` continues to carry the same value for backward compatibility (existing dedup keeps working).
2. Add an `event_class` metadata entry with value `expiring_subscription` so dispatcher can cheaply decide whether the dismissal cache is even relevant.
3. Update the events.processor / dispatcher recipient filter to read `Metadata["subscription_id"]` (preferred) and fall back to `Location` if absent — guarantees safe rollout against in-flight events.

**Resulting normalized key**

```
dismissal_key = (user_id, subscription_id)
            where subscription_id = License.ID.String()  -- UUID, lower-case canonical form
```

This is the key used by:

- The PRIMARY KEY of `subscription_dismissals` (§4.1).
- The dismissal cache in dispatcher and processor.
- The JWT `subscription_id` claim in the email dismiss token (§5.4).
- The audit-log entry (NFR-3).

**Display vs identity**

The UI and email templates show the human-friendly **SKU + account name** (the renderer already has `skuName` and `account_name` in scope). The Dismiss button/link always carries the underlying `License.ID` — never the SKU — so dismissing one expiring license does not silence a different license that happens to share the SKU.

**Renewal interaction**

When license-service publishes a `CLEARED` event for a renewed license (already implemented in `RunExpiredHandler`), the new replacement license has its own fresh `License.ID`. The previous dismissal is bound to the old `License.ID` and is therefore correctly *not* applied to the renewal — the user will see the next expiry warning when that new license eventually approaches end-of-term, which is the desired behavior.

---

## 4. Data model

### 4.1 New table — `subscription_dismissals` (mailbox DB)

```sql
CREATE TABLE subscription_dismissals (
    user_id          TEXT        NOT NULL,                 -- Athena user identifier
    subscription_id  UUID        NOT NULL,                 -- = athena.license.service License.ID
    account_id       TEXT        NOT NULL,                 -- numeric account id, stored as text for parity with notifications stack
    scope            TEXT        NOT NULL DEFAULT 'subscription',
    source_channel   TEXT        NOT NULL,                 -- 'in_app' | 'email'  -- where the user clicked Dismiss (audit only)
    dismissed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, subscription_id)
);

CREATE INDEX ix_dismissals_account      ON subscription_dismissals (account_id);
CREATE INDEX ix_dismissals_subscription ON subscription_dismissals (subscription_id);
```

- `scope` is reserved for future expansion (e.g. `type`, `category`) — keeps the API stable.
- `source_channel` is **descriptive only** — it records *where the dismiss action originated* for audit/analytics. **It does NOT scope the suppression**: see §4.1.1.
- Cascade delete is handled via account/user lifecycle hooks (existing pattern in mailbox).

### 4.1.1 Channel semantics — one row = suppression on **all** channels

Notifications for an expiring subscription are delivered today on two user-facing channels:

| Channel | Owner | Where suppression is enforced |
|---|---|---|
| **`in_app`** (mailbox / portal) | [atlas.notifications.mailbox](../../atlas.notifications.mailbox) | (a) read-path filter on `GET /v1/mailbox/notifications` and (b) write-path skip in the mailbox Kafka consumer |
| **`email`** | [atlas.notifications.postoffice](../../atlas.notifications.postoffice) (via [atlas.notifications.dispatcher](../../atlas.notifications.dispatcher) recipient resolution) | recipient drop in dispatcher *and* hard skip in postoffice email renderer |

**Rule (FR-3, FR-8):** a single `subscription_dismissals` row for `(user_id, subscription_id)` permanently suppresses **every channel** for that pair. There is no "dismiss email only" or "dismiss in-app only" mode in 26.4.

Consequences:

- The dismiss API request body **does not accept a channel field**. The button in the portal and the link in the email both produce identical suppression.
- `source_channel` on the table is `in_app` when the user clicked the in-app Dismiss button, `email` when they used the tokenized link, and `admin` for the support override (§5.1). It is metadata, not a filter key.
- Dispatcher's recipient-drop logic and mailbox's read/write filters share **one cache lookup**: `IsDismissed(user_id, subscription_id) bool`. They never branch on `source_channel`.
- Future per-channel granularity ("silence email but keep in-app") would be a new feature: it would change the PK to `(user_id, subscription_id, channel)`, add `channel` to the cache key, and require a UI change. Out of scope for PTCI-4043.

Why not store `channel = 'all' | 'in_app' | 'email'` today?

- The FS explicitly demands suppression on **every** channel (FR-3, FR-8). A per-channel column would introduce ambiguity ("is `'all'` a third value, or the absence of `'in_app'`/`'email'` rows?") with no functional benefit.
- Keeping the PK at `(user_id, subscription_id)` makes the cache simpler and guarantees the FR-9 "per-user" invariant via the database, not application code.

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
  "subscription_id": "<License.ID UUID>"
}

201 Created
{
  "user_id":         "...",
  "subscription_id": "<License.ID UUID>",
  "account_id":      "...",
  "dismissed_at":    "2026-04-28T15:17:09Z",
  "source_channel": "in_app",          // audit only — suppression applies to in_app + email
  "suppresses":      ["in_app", "email"]
}
```

Request body intentionally omits any channel field — see §4.1.1 (one dismiss = all channels). The response includes `suppresses` so clients have an explicit, forward-compatible enumeration.

```
GET  /v1/mailbox/dismissals
DELETE /v1/mailbox/dismissals/{subscription_id}     # not exposed in UI; admin-only
```

### 5.2 Mailbox — service-to-service

```
POST /v1/internal/mailbox/dismissals
Authorization: Bearer <S2S JWT>
{
  "user_id":         "...",
  "subscription_id": "<License.ID UUID>",
  "account_id":      "...",
  "source_channel": "email"           // 'in_app' | 'email' | 'admin' — audit only
}
```

Used by postoffice after token verification; uses Athena S2S claims (existing pattern).

### 5.2.1 Idempotency & concurrency guarantees

Both write endpoints (`POST /v1/mailbox/dismissals` and `POST /v1/internal/mailbox/dismissals`) are **idempotent on `(user_id, subscription_id)`** and **safe under arbitrary retries** by client, gateway, or Kafka redelivery.

**Contract**

| Aspect | Behavior |
|---|---|
| Idempotency key | `(user_id, subscription_id)` — the table PRIMARY KEY (§4.1) |
| First successful call | `201 Created` + dismissal row written |
| Repeat call (same user+sub) | `200 OK` + the existing row returned unchanged |
| `dismissed_at` | Set on first write only; **never updated** by retries |
| `source_channel` | Set on first write only; **never updated** by retries — preserves true audit origin |
| Concurrent first writes | Exactly one row created via `INSERT ... ON CONFLICT (user_id, subscription_id) DO NOTHING RETURNING *` + `RETURNING`-coalesce read; other callers observe `200 OK` with the winning row |
| `dismissal.changed` Kafka event | Emitted **only** on actual insert (`xmax = 0` check); duplicate writes do **not** re-emit |
| Audit log | One entry per *successful insert*; retries log a `dismiss_retry_noop` debug line, not a duplicate audit row |
| HTTP error semantics | `4xx` → safe to retry; `5xx` after commit → safe to retry (next call returns `200 OK`); never produces partial state |

**SQL pattern (mailbox)**

```sql
WITH ins AS (
    INSERT INTO subscription_dismissals
        (user_id, subscription_id, account_id, scope, source_channel)
    VALUES ($1, $2, $3, 'subscription', $4)
    ON CONFLICT (user_id, subscription_id) DO NOTHING
    RETURNING *, true AS inserted
)
SELECT * FROM ins
UNION ALL
SELECT *, false AS inserted FROM subscription_dismissals
WHERE user_id = $1 AND subscription_id = $2
LIMIT 1;
```

The single-statement pattern guarantees:

- No `SELECT-then-INSERT` race between concurrent callers.
- The handler reads `inserted` to decide whether to publish the Kafka event and emit the audit log.
- The transaction also performs the in-app archive update (§6.1 step 4) and the Kafka publish through the **outbox table** (existing pattern in mailbox), so the dismissal row + audit + Kafka event are committed atomically. Retries that observe `inserted=false` skip both the archive update and the Kafka publish, since the original commit already produced them.

**Email-channel implications (NFR-1 interaction)**

The dismiss-token JTI is consumed in Redis **before** the S2S call to mailbox (§6.2 step 4). Therefore:

- A redelivery of the *same* email click with the *same* token → blocked at the JTI check (`410 Gone`); never reaches mailbox.
- A retry of the S2S call inside postoffice (e.g. transient 5xx) → reuses the JTI logically by skipping the `SET NX` step on its own retry path; mailbox is also idempotent so a duplicated S2S call is harmless and yields `200 OK`.
- A user who clicks an *email link*, then *also* dismisses from the portal → second call is a no-op; row already exists.

**Read endpoints**

`GET /v1/mailbox/dismissals` and `GET /v1/internal/mailbox/dismissals/{user}/{sub}` are inherently idempotent and side-effect-free.

**Delete / undismiss (admin)**

`DELETE /v1/mailbox/dismissals/{subscription_id}` is idempotent: missing row → `204 No Content`, present row → `204 No Content` after delete. Emits an `undismissed` Kafka event only when a row was actually removed.

---

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
   - inserts/`ON CONFLICT DO NOTHING` into `subscription_dismissals` (`source_channel='in_app'`),
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
   - call mailbox `POST /v1/internal/mailbox/dismissals` with `source_channel="email"`,
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
  "user_id":         "...",
  "subscription_id": "<License.ID UUID>",
  "account_id":      "...",
  "source_channel": "in_app" | "email" | "admin",   // audit only
  "ts": "..."
}
```

Dispatcher (`pkg/mq/handler_dismissals_changes.go`) and processor consume this and update their in-memory caches (TTL fallback 5 min).

### 6.6 Cache-invalidation race (cold-cache window)

There is an inherent gap between *(a)* mailbox committing the dismissal row and *(b)* dispatcher / events.processor receiving the `dismissal.changed` Kafka event and updating their local caches. In practice this gap is sub-second (Dapr/Kafka in-cluster), but the worst case is bounded by Kafka consumer lag and pod cold-start.

#### 6.6.1 The race

```
t0   user clicks Dismiss in portal
t0+ε mailbox commits row, emits dismissal.changed, returns 201
t1   RunExpiredHandler in athena.license.service publishes
       entitlement-expiry-warning for the same License.ID
t2   dispatcher resolves recipients; cache still cold for this user
       → dispatcher does NOT drop the user
       → in-app + email warning slip through  ❌
t3   dispatcher consumes dismissal.changed, cache now hot
       (all subsequent events suppressed correctly ✅)
```

The race window is `t3 − t0`. During that window **one last warning may be delivered** even though the user has already dismissed the subscription.

#### 6.6.2 Why the cron schedule makes this unlikely-but-not-impossible

`RunExpiredHandler` runs as the `entitlement-expired-notification` CronJob (daily — see [athena.license.service/site/content/operator/getting-started.md](../../athena.license.service/site/content/operator/getting-started.md#L58)). Inside one cron run it iterates every account and emits one event per qualifying license. If the user clicks Dismiss while that cron is *currently iterating* their account, dispatcher's cache may not yet be primed when the recipient resolution for *their* event happens a few hundred milliseconds later. The window is small but real.

#### 6.6.3 Decision: **accept eventual consistency for general events; add read-through-on-miss for expiring-subscription events only**

The platform-wide cache invalidation pattern (filters, subscriptions) is eventually consistent and the team has accepted that for high-volume general notifications. We diverge from it **only** for the expiring-subscription flow because:

- Volume is tiny (single-digit dismissals per user per year, daily cron).
- A "one last email after I dismissed" experience is exactly the alert-fatigue problem this feature was created to solve. Letting it slip through partially defeats the purpose.
- The check is cheap: at most one indexed lookup keyed by `(user_id, subscription_id)`, only on the dismissal-relevant branch.

**Mechanism**

In dispatcher and events.processor, the dismissal cache becomes a **negative-result cache with a short TTL plus read-through on miss**, but **only for the `expiring_subscription` event class**:

```go
// pseudo-code, dispatcher recipient filter
func (c *DismissalCache) IsDismissed(ctx context.Context, eventClass, userID string, subID uuid.UUID) bool {
    if eventClass != "expiring_subscription" {
        return c.lookupHotOnly(userID, subID)            // existing fast path
    }
    if v, ok := c.lookupHotOnly(userID, subID); ok {
        return v                                          // hit (positive or negative)
    }
    // Cold: read-through to mailbox source of truth.
    dismissed, err := c.mailboxClient.IsDismissed(ctx, userID, subID)
    if err != nil {
        // fail-open is safe (we may send one extra warning, never silence wrongly)
        c.metrics.IncReadThroughError()
        return false
    }
    c.put(userID, subID, dismissed, negativeTTL)         // 30s neg TTL
    return dismissed
}
```

Properties:

- **Bounded extra load** — the read-through only fires for cache misses on expiring-subscription events. Hit ratio after warm-up is ~100 %; cold misses are at most one per `(user, subscription)` per pod lifetime.
- **Short negative TTL (30 s)** caps the staleness window for dismissals that fire *after* a negative cache entry was filled. A subsequent `dismissal.changed` event still invalidates earlier than the TTL.
- **Fail-open** on read-through error — in the worst case the user sees one extra warning, never the inverse (a wrongful silence on an undismissed subscription). This matches the safety bias of the rest of the notifications stack.
- The non-expiry hot path is **unchanged**, so we don't pay the read-through cost on the high-volume general traffic.

**Mailbox `IsDismissed` lookup endpoint**

```
GET /v1/internal/mailbox/dismissals/{user_id}/{subscription_id}
  → 200 { "dismissed": true,  "dismissed_at": "..." }
  → 200 { "dismissed": false }
```

S2S only. Single indexed PK lookup. Cached at the dispatcher side per the rules above.

#### 6.6.4 Optional source-side guard

`RunExpiredHandler` is the only producer of expiring-subscription events today. As an additional safety net we may, in a follow-up story (not required for 26.4), have it call mailbox's `IsDismissed` before publishing each event. This shrinks the window further and lets us skip publishing entirely for dismissed `(user, sub)` pairs. Trade-off: one extra S2S call per license per cron tick \u2014 acceptable given the cron is daily and the license set per account is bounded.

#### 6.6.5 Observability for the race

- `dismissals_cache_readthrough_total{result="hit_dismissed"|"hit_active"|"error"}`
- `notifications_suppressed_total{reason="dismissed", layer="dispatcher", path="readthrough"|"cache"}`
- `dismissal_invalidation_lag_seconds` \u2014 histogram of `(dispatcher_apply_ts - mailbox_commit_ts)` derived from Kafka headers; alert if `p99 > 5s`.
- A single counter `expiry_warning_sent_after_dismiss_total` populated by mailbox when it later detects a write attempt for an already-dismissed `(user, sub)` (best-effort signal that a slip-through actually occurred).

#### 6.6.6 Acceptance

With (a) Kafka-driven invalidation, (b) read-through-on-miss for expiring-subscription only, and (c) optional source-side guard in a follow-up, the probability of a "one last warning after dismiss" is reduced to:

- **Zero** for in-app notifications, because mailbox is the source of truth and its read/write paths consult `subscription_dismissals` directly (no cache for the authoritative writer).
- **Near-zero** for email, because dispatcher will read-through on the first cold miss, and the daily cron cadence makes a *second* expiry email in the same window structurally impossible.

This is documented as the explicit, accepted behaviour for FR-3 / FR-8.

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

- `dismissals_created_total{source_channel="in_app"|"email"|"admin"}`
- `dismissals_undone_total{actor="admin"}`
- `notifications_suppressed_total{reason="dismissed", layer="mailbox"|"dispatcher"|"postoffice", channel="in_app"|"email"}`
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
