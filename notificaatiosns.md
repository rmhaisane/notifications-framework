# Part 1 — How the notification system works **today** (in detail, simple words)

Think of the CSP notifications system like a small **post office** with several specialised workers. Each worker has one job, and they pass letters to each other on a conveyor belt called **Kafka**.

## The cast of characters

| Service (the "worker") | What it does, in one sentence |
|---|---|
| **athena.license.service** | The **calendar keeper**. Knows every customer's license: when it starts, when it expires, when it enters grace period. |
| **atlas.notifications.events.processor** | The **translator + scheduler**. Turns raw license/system events into normalised "notification events" and decides how often to remind. |
| **atlas.notifications.config** | The **rule book**. Stores triggers ("which events matter"), channels (email, in-app, webhook, Slack, PagerDuty), templates (what the email looks like), and per-account subscriptions ("send these events to these emails through this template"). |
| **atlas.notifications.dispatcher** | The **router**. Reads the rule book, matches incoming events to the right recipients and channels. |
| **atlas.notifications.sender** | The **delivery van**. Actually sends the email/webhook/PagerDuty/Slack message. |
| **atlas.notifications.mailbox** | The **portal inbox**. Stores in-app alert cards (the bell icon) so users can read them in the CSP portal. |
| **CSP Portal (frontend)** | What the user actually sees. Reads from mailbox, shows the bell, lists alerts. |

## What happens when your subscription is about to expire

### Step 1 — The calendar keeper notices
The license service has a scheduler. For each license it stores two future "alarms":
- `license.start` — when the license begins.
- `license.expiry` — when it expires (plus grace period).

When the alarm time arrives — or when an admin manually changes/removes a license — the license service builds a small JSON message:

```json
{ "account_id": "...", "sfdc_account_id": "...", "license_id": "...",
  "start_date": "...", "end_date": "...", "type": "EXPIRED" }
```

…and **publishes it to Kafka** through Dapr (`SendNotification` in notifications.go).

### Step 2 — The translator picks it up
The events processor consumes that Kafka topic. It does three useful things:

1. **Normalises** the message into a standard "notification event" with a `event_type=ACCOUNT` and a `event_subtype` like `entitlement_expired` or `entitlement_entered_grace_period`.
2. **Summarises** so we don't spam users — e.g. the *License Expiration* summarisation rule with a 5-minute time-cycle (0034_summarization_rules.up.sql).
3. **Schedules reminders** — controlled by `event.reminder.expiration.multiple` (event_usecase.go). This is **why you keep getting reminded** — the processor keeps re-emitting until the event "expires" from the reminder schedule.

It then publishes this clean event back onto Kafka.

### Step 3 — The event splits into TWO roads

This is the most important architectural point, and it's relatively recent — until not long ago, **everything went through the sender**, including in-app. Now the in-app road bypasses sender:

```
                                     ┌──────────────────────────────────────┐
                                     │   events.processor → Kafka topic     │
                                     └──────────────┬───────────────────────┘
                                                    │
                ┌───────────────────────────────────┼───────────────────────────────┐
                │                                                                   │
        ROAD A (in-app)                                                ROAD B (email/webhook/etc.)
        NEW — direct consumer flow                                     LEGACY — through dispatcher+sender
                │                                                                   │
                ▼                                                                   ▼
   atlas.notifications.mailbox                                        atlas.notifications.dispatcher
   pkg/consumer/index.go (Init when                                   matches event vs. cached
   consumer.flow.enabled=true)                                        triggers/subscriptions/filters
   → channels/inapp.go                                                (pkg/cache/cache_*.go,
   → channels/notifier.go                                             pkg/mq/handler_*_changes.go)
   writes account_alerts / user_alerts                                                │
   (with SHA-256 idempotency key                                                      ▼
   so Kafka replays don't duplicate                                  atlas.notifications.sender
   the card — see pkg/pb/service.proto)                              renders template, calls SMTP/
                                                                     webhook/PagerDuty/Slack
```

### Road A — How the bell-icon card appears
1. Mailbox is **subscribed directly to the Kafka topic** via Dapr (pkg/consumer/index.go).
2. For each event, channels/inapp.go builds the card text.
3. channels/notifier.go writes a row into either `user_alerts` (per-user) or `account_alerts` (account-wide).
4. A **SHA-256 idempotency key** is stored on the row so if Kafka redelivers the same message the row isn't duplicated.
5. The portal reads from the gRPC/REST API exposed by pkg/svc/account.alerts.server.go and shows the cards.
6. The portal's existing "Dismiss" simply flips `state` to `dismissed` (3) on **that one card** (mapping.go). After `archive.after` time it's archived (archive.go).

### Road B — How the email arrives
1. Dispatcher keeps an in-memory cache of the rule book (subscriptions, channels, triggers, filters), refreshed via Kafka "*_changes" topics (handler_subscription_changes.go, etc.).
2. When the event matches a subscription whose channel is `INFOBLOX_EMAIL`, dispatcher hands the rendered payload to **sender**.
3. Sender renders the template + recipient list and pushes it out (SMTP / SES / etc., depending on env).

### Why users feel "spammed"
- The reminder loop in events processor re-publishes the event on a schedule.
- Each re-publish → new card in mailbox + new email through sender.
- The current "Dismiss" only kills the **single card**, not the recurring schedule.
- There is **no concept anywhere** of *"this user already said: stop bugging me about this subscription."*

That's exactly the gap the story is asking us to close.

---

# Part 2 — What needs to change (in detail, simple words)

The fix is conceptually small: **add a memory** of "user U muted subscription S", and **make every place that creates a notification consult that memory first**. The complication is that "every place" is now **two services** (mailbox for in-app, dispatcher for email/webhook), because of the new consumer flow.

## Change 1 — A new "mute list" table (the memory)

New table `notification_suppressions` in atlas.notifications.config (it already owns rules/triggers/subscriptions, so this fits naturally):

| column | meaning |
|---|---|
| `id` | row id |
| `user_id` | the **person** who muted |
| `account_id` | their account (for query/audit) |
| `subscription_id` | the **license/entitlement id** they muted |
| `event_subtype` | e.g. `entitlement_expired`, `entitlement_entered_grace_period` (so we can mute "expiry stuff" but still get other event types) |
| `source` | `portal` or `email` (where the click came from) |
| `created_at` | when |
| unique index | `(user_id, subscription_id, event_subtype)` so duplicates can't happen |

Plus a small `dismissal_events` audit table (or write to the existing audit pipeline).

## Change 2 — A small CRUD API on top

In atlas.notifications.config:
- `POST /v1/notification_suppressions` — "mute this for me" (caller must be the same user, enforced by JWT).
- `GET /v1/notification_suppressions` — "what have I muted" (used by portal "manage notifications" screen).
- `DELETE /v1/notification_suppressions/{id}` — admin/support un-mute.
- `POST /v1/notification_suppressions/by_token` — used by the email link (no JWT, the token *is* the auth).
- `GET /v1/notification_suppressions/preview/{token}` — email landing page calls this to show *which* subscription is about to be muted before user clicks confirm.

## Change 3 — The email gets a "Dismiss" link

- Update the entitlement-expiry **template** in `notification_templates` (in config DB) to include something like:
  > *Don't warn me again about this subscription:* `https://csp.infoblox.com/notifications/dismiss?token=<JWT>`
- The token is a **signed JWT** containing `user_id`, `subscription_id`, `event_subtype`, `exp` (default 7 days, configurable), and a `jti` (single-use id).
- Token minting happens in config when the email is being prepared (sender asks config for the URL, or config injects it before handing the rendered body to sender).
- When the user clicks: portal opens the dismiss page, calls `preview/{token}` to show details, then `by_token` to commit. The `jti` is recorded so the same link can't be reused.

This satisfies **NFR-1 (security)**: time-limited, single-use, signed, no PII in the URL.

## Change 4 — The portal gets a "Dismiss" button + confirmation modal

On the bell-icon card for any expiry alert:
- A "Dismiss" button.
- Click → modal:
  > *"This will permanently hide all future expiration warnings for **<subscription name>**. This action cannot be undone."*
  > [ Cancel ] [ **Yes, permanently dismiss all future warnings** ]
- Confirm → calls `POST /v1/notification_suppressions`.
- On success:
  - Remove the card immediately from the UI (optimistic update).
  - Show toast: *"Future warnings for <subscription> are suppressed."*
  - Trigger a background mark-as-dismissed for any other open cards for the same `(user, subscription, expiry-class)`.

## Change 5 — Both roads must check the mute list (the architecturally important one)

Because in-app and email take different roads now, we need **two enforcement points** that consult the **same** source of truth.

### 5a — In atlas.notifications.mailbox (in-app road)
- In `channels/notifier.go` (or just before the `INSERT INTO user_alerts`), look up suppression.
- If suppressed → skip the insert, **still ack the Kafka offset** (otherwise we'd be stuck), and emit a metric `notifications_suppressed_total{channel="inapp"}`.
- Also retroactively mark any existing not-yet-dismissed `user_alerts` rows for the same `(user, subscription, subtype)` as `dismissed` so they disappear immediately (FR-4).

### 5b — In atlas.notifications.dispatcher (email/webhook road)
- New cache module `pkg/cache/cache_suppressions.go`, mirroring the existing pattern in pkg/cache/cache_subscriptions.go.
- In pkg/mq/handler_notification_delivery.go, filter out recipients whose `(user_id, subscription_id, event_subtype)` is in the cache before handing to sender.
- Cache invalidation via a new Kafka topic `suppression_changes` (mirroring the existing `subscription_changes` / `filter_changes` topics — see handler_subscription_changes.go).

### 5c — Why a Kafka "suppression_changes" topic?
- Both services are already wired to consume "*_changes" topics for cache invalidation.
- Reuses existing patterns and ops tooling.
- A click in the portal → config writes the row → publishes a small change message → both mailbox and dispatcher caches refresh in seconds → no more emails, no more cards.

## Change 6 — Audit logging

Every create / token-redeem / admin-delete writes one audit row:
`{ user_id, subscription_id, channel="portal"|"email", source_ip, user_agent, ts, action }`

Satisfies **NFR-3 (auditability)**.

## Change 7 — Safety, observability, ops

- **Metrics** (Prometheus, on every service):
  - `dismissals_total{source}` (config)
  - `notifications_suppressed_total{channel,event_subtype}` (mailbox & dispatcher)
  - `dismiss_token_invalid_total{reason="expired|used|signature"}` (config) — alert if it spikes (possible abuse).
- **Feature flag** in featureflag to roll out: dev → staging → prod.
- **Support tooling**: a small script under `aide/` to list/delete a user's suppression on customer request.
- **Sender legacy in-app path** (inapp.go): audit it; if it can still be reached when `consumer.flow.enabled=false`, gate it with the same suppression check so a flag flip can't cause a regression.

---

## What "good" looks like at the end

A user clicks **Dismiss** on an expiry card:
1. Portal calls config; row is written; audit row recorded; `suppression_changes` Kafka message fires.
2. Mailbox cache + dispatcher cache update within seconds.
3. The clicked card disappears immediately; any sibling cards for the same subscription are flipped to `dismissed`.
4. The next reminder cycle from events processor still produces the event, but:
   - **Mailbox** sees it, checks the cache, does **not** insert into `user_alerts`.
   - **Dispatcher** sees it, checks the cache, does **not** ask sender to send the email.
5. A user clicks **Dismiss** in the email instead → identical end result, via the signed token + landing page.
6. Other users on the same account are completely unaffected (FR-9).

That, end-to-end, is the change.
