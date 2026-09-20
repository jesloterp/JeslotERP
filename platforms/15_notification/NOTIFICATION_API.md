# JeslotERP Notification Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** Enterprise API baseline (pre-implementation) — output / notification control plane  
**Package:** `platforms.p15_notification`  
**PostgreSQL schema:** `notification`  
**Public base:** `/api/v1/notifications`  
**Internal base:** `/internal/v1/notifications`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`NOTIFICATION_GUIDE.md`](NOTIFICATION_GUIDE.md) · [`NOTIFICATION_SCHEMA.md`](NOTIFICATION_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Send intents, prefs/inbox, templates, routing, deliveries, suppressions, digests, provider webhooks, packs. |

---

## 1. Design principles (advanced)

1. **Send intent first** — `POST /send` accepts; dispatch is async.  
2. **Prefs & suppression before render** when possible.  
3. **Idempotent send** — same key → same request.  
4. **Critical topics** require `notify.send.critical` to bypass quiet hours.  
5. **No secrets in responses** — providers show `secret_ref_key` only.  
6. **Inbox is a channel** — always creatable for user recipients when routed.  
7. **Locale explicit or resolved** — never random.  
8. **Template ACTIVE pin** at accept time.  
9. **Provider I/O via p14** — HTTP APIs don’t block on SMTP.  
10. **Webhook signatures verified**.  
11. **Dry-run / preview** without send.  
12. **PII-aware** delivery inspection.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
X-Correlation-Id: <uuid>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…" }
}
```

---

## 4. Errors

```text
NTF_TOPIC_NOT_FOUND / TEMPLATE_NOT_FOUND / TEMPLATE_NOT_ACTIVE
NTF_CHANNEL_DENIED / ROUTE_EMPTY
NTF_RECIPIENT_INVALID / CONTACT_UNVERIFIED
NTF_PREFERENCE_SUPPRESSED / QUIET_HOURS / DND_ACTIVE
NTF_SUPPRESSED_ADDRESS / UNSUBSCRIBED
NTF_MERGE_INVALID / RENDER_FAILED
NTF_PROVIDER_UNAVAILABLE / PROVIDER_REJECTED
NTF_ATTACHMENT_INVALID / MEDIA_NOT_AVAILABLE
NTF_IDEMPOTENCY_CONFLICT
NTF_INBOX_NOT_FOUND
NTF_WEBHOOK_SIGNATURE_INVALID
NTF_DIGEST_NOT_ALLOWED
NTF_APPROVAL_REQUIRED / PACKAGE_CHECKSUM_MISMATCH
NTF_CRITICAL_DENIED
NTF_RATE_LIMITED
```

HTTP: `404` · `409` · `422` · `403` · `429`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `notify.template.read` / `manage` | Templates |
| `notify.send` | Send intents |
| `notify.send.critical` | Critical/bypass |
| `notify.prefs.manage` | Admin prefs |
| `notify.inbox.read` | Inbox (own by default) |
| `notify.delivery.read` | Deliveries |
| `notify.provider.manage` | Providers |
| `notify.suppress.manage` | Suppressions |
| `notify.pack.install` | Packs |
| `notify.audit.read` | Audit |
| `notify.*` | All |

---

## 6. Send APIs (primary)

### 6.1 Send notification

```http
POST /api/v1/notifications/send
Idempotency-Key: …
```

```json
{
  "topic_key": "workflow.task",
  "template_key": "notify.process.task_assigned",
  "priority": "HIGH",
  "locale": "hi-IN",
  "recipients": [
    { "user_id": "…" }
  ],
  "data": {
    "task_title": "Approve sales order cancel",
    "process_key": "sales.order.cancel.approval",
    "deep_link": "/inbox/tasks/…"
  },
  "channels": null,
  "route_policy_key": "default.user",
  "attachments": [{ "media_id": "…" }],
  "scheduled_for": null
}
```

**Response:**

```json
{
  "request_id": "…",
  "status": "ACCEPTED",
  "deliveries": [
    { "delivery_id": "…", "channel": "INAPP", "status": "QUEUED" },
    { "delivery_id": "…", "channel": "EMAIL", "status": "QUEUED" },
    { "delivery_id": "…", "channel": "PUSH", "status": "DEFERRED" }
  ]
}
```

Statuses may be `SUPPRESSED` / `DEFERRED` (quiet hours) at accept time.

### 6.2 Send critical (OTP)

```http
POST /api/v1/notifications/send-critical
Idempotency-Key: …
```

Requires `notify.send.critical`. Forces topic policy channels (e.g. SMS).

### 6.3 Preview / dry-run

```http
POST /api/v1/notifications/preview
```

```json
{
  "template_key": "notify.process.task_assigned",
  "locale": "hi-IN",
  "channel": "EMAIL",
  "data": { "task_title": "…" }
}
```

Returns rendered subject/body; no ledger write.

### 6.4 Get request / deliveries

```http
GET /api/v1/notifications/requests/{request_id}
GET /api/v1/notifications/requests/{request_id}/deliveries
GET /api/v1/notifications/deliveries/{delivery_id}
GET /api/v1/notifications/deliveries/{delivery_id}/events
```

### 6.5 Cancel deferred

```http
POST /api/v1/notifications/requests/{request_id}/cancel
```

Only if not fully dispatched.

---

## 7. In-app inbox (primary for users)

```http
GET  /api/v1/notifications/inbox
GET  /api/v1/notifications/inbox/unread-count
GET  /api/v1/notifications/inbox/{message_id}
POST /api/v1/notifications/inbox/{message_id}/read
POST /api/v1/notifications/inbox/read-all
POST /api/v1/notifications/inbox/{message_id}/archive
DELETE /api/v1/notifications/inbox/{message_id}
```

Query: `status=UNREAD&topic_key=workflow.task`.

---

## 8. Preferences & devices

### 8.1 My preferences

```http
GET /api/v1/notifications/preferences/me
PUT /api/v1/notifications/preferences/me
GET /api/v1/notifications/quiet-hours/me
PUT /api/v1/notifications/quiet-hours/me
```

**Put prefs:**

```json
{
  "items": [
    { "topic_key": "workflow.task", "channel": "EMAIL", "is_enabled": true, "digest_mode": "INSTANT" },
    { "topic_key": "workflow.task", "channel": "SMS", "is_enabled": false }
  ]
}
```

Security topics may be non-disableable.

### 8.2 Admin prefs

```http
GET /api/v1/notifications/preferences/users/{user_id}
PUT /api/v1/notifications/preferences/users/{user_id}
PUT /api/v1/notifications/tenants/defaults
```

### 8.3 Devices (push)

```http
POST /api/v1/notifications/devices
DELETE /api/v1/notifications/devices/{token_id}
GET /api/v1/notifications/devices/me
```

```json
{
  "platform": "android",
  "token": "…",
  "app_id": "jesloterp"
}
```

### 8.4 Unsubscribe (token)

```http
POST /api/v1/notifications/unsubscribe
GET  /api/v1/notifications/unsubscribe/{token}   # landing support
```

Body includes opaque token; server hashes & suppresses.

---

## 9. Templates (admin)

```http
GET    /api/v1/notifications/templates
POST   /api/v1/notifications/templates
GET    /api/v1/notifications/templates/{template_key}
GET    /api/v1/notifications/templates/{template_key}/versions
POST   /api/v1/notifications/templates/{template_key}/versions
PUT    /api/v1/notifications/versions/{version_id}/locales/{locale}/channels/{channel}
POST   /api/v1/notifications/versions/{version_id}/publish
POST   /api/v1/notifications/versions/{version_id}/activate
POST   /api/v1/notifications/versions/{version_id}/simulate
```

**Locale channel put:**

```json
{
  "subject_icu": "Task assigned: {task_title}",
  "body_html_icu": "<p>You have a task: {task_title}</p>",
  "body_icu": "You have a task: {task_title}"
}
```

---

## 10. Topics, routes, providers

```http
GET  /api/v1/notifications/topics
POST /api/v1/notifications/topics
PUT  /api/v1/notifications/topics/{topic_key}/policy

GET  /api/v1/notifications/route-policies
POST /api/v1/notifications/route-policies
PUT  /api/v1/notifications/route-policies/{key}/steps

GET  /api/v1/notifications/providers
POST /api/v1/notifications/providers
POST /api/v1/notifications/providers/{provider_key}/test
GET  /api/v1/notifications/providers/{provider_key}/health
```

**Route steps example:**

```json
{
  "steps": [
    { "position": 1, "channel": "PUSH", "optional": true },
    { "position": 2, "channel": "INAPP", "optional": false },
    { "position": 3, "channel": "EMAIL", "optional": true },
    { "position": 4, "channel": "SMS", "optional": true, "on_failure_of": ["EMAIL"] }
  ]
}
```

---

## 11. Suppressions & digests

```http
GET  /api/v1/notifications/suppressions
POST /api/v1/notifications/suppressions
DELETE /api/v1/notifications/suppressions/{id}

GET  /api/v1/notifications/digests/policies
PUT  /api/v1/notifications/digests/policies/{key}
POST /internal/v1/notifications/digests/flush
```

Flush builds digest emails from buckets (scheduler-triggered).

---

## 12. Provider webhooks (inbound)

```http
POST /api/v1/notifications/webhooks/{provider_key}
POST /internal/v1/notifications/webhooks/{provider_key}
```

Verifies signature → updates delivery (delivered/bounce/complaint/open/click) → may create suppression.

---

## 13. Internal dispatch

```http
POST /internal/v1/notifications/dispatch/{delivery_id}
POST /internal/v1/notifications/send
```

- `dispatch` invoked by p14 handler `notify.dispatch_channel`  
- internal `send` for trusted platforms (identity OTP, process) with service auth  

---

## 14. Packages & governance

```http
GET  /api/v1/notifications/packages
POST /api/v1/notifications/packages/{package_key}/install
GET  /api/v1/notifications/changesets
POST /api/v1/notifications/changesets/{id}/approvals
```

---

## 15. Audit & stats

```http
GET /api/v1/notifications/stats?from=…&to=…&channel=EMAIL
GET /api/v1/notifications/audit/requests?topic_key=…
```

---

## 16. Caching & concurrency

| Resource | Strategy |
|---|---|
| ACTIVE templates | Cache by template_key+locale+channel |
| Prefs | Cache per user; invalidate on put |
| Suppressions | Hot cache by address hash |
| Send idempotency | Durable unique key |
| Provider health | Short TTL + failover |

---

## 17. Example flows

### 17.1 Task assigned

1. p10 task.created → bridge/internal send  
2. Prefs → INAPP+EMAIL+PUSH  
3. Jobs dispatch email/push; inbox row created  
4. User reads inbox → read API  

### 17.2 OTP

1. Identity `send-critical` topic `security.otp`  
2. Forced SMS; quiet hours bypassed  
3. Delivery SENT; short TTL body  

### 17.3 Quiet hours

1. Normal email at 23:30 local → DEFERRED  
2. Scheduler/worker releases after quiet end  

### 17.4 Bounce

1. SES webhook hard bounce  
2. Suppression created  
3. Future sends skip email channel  

---

## 18. Event hooks

| Event | Consumer |
|---|---|
| `notify.delivery.failed` | Ops / fallback already applied |
| `notify.provider.unhealthy` | Page on-call |
| `notify.inbox.created` | Optional websocket fanout |
| `notify.preference.changed` | Analytics |

---

## 19. Compatibility notes

- Public prefix `/api/v1/notifications`; schema `notification`.  
- Prefer template keys over ad-hoc HTML from callers.  
- Attachments must be AVAILABLE (not quarantined) media.  
- WhatsApp outbound often requires pre-approved provider templates — map via `ntf_whatsapp_mapping`.

---

## 20. Related documents

- Guide: [`NOTIFICATION_GUIDE.md`](NOTIFICATION_GUIDE.md)  
- Schema: [`NOTIFICATION_SCHEMA.md`](NOTIFICATION_SCHEMA.md)  
- Localization: [`../06_localization/LOCALIZATION_API.md`](../06_localization/LOCALIZATION_API.md)  
- Messaging: [`../14_messaging/MESSAGING_API.md`](../14_messaging/MESSAGING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
