# JeslotERP Notification Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — SMTP/SMS HTTP/WhatsApp Graph clients are wired; pytest stays stub; empty creds return `PROVIDER_PENDING` (never invent a vendor id). Not Production.  
**Package:** `platforms.p15_notification`  
**PostgreSQL schema:** `notification`  
**Depends on:** `p01_identity`, `p06_localization`, `p14_messaging`  
**Integrates with:** `p02_organization`, `p03_configuration` (provider secrets), `p08_file_media` (attachments), `p09_document`, `p10_process` (task alerts), `p11_rules` (routing), `p12_feature`, `p13_event_bus`, `p17_scheduler` (digests)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`NOTIFICATION_SCHEMA.md`](NOTIFICATION_SCHEMA.md) · [`NOTIFICATION_API.md`](NOTIFICATION_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full omnichannel notification plane: templates+i18n, channels, preferences/consent, quiet hours, routing/fallback, delivery ledger, providers, digests, in-app inbox, bounces, throttling, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-005: live SMTP/SMS/WhatsApp ports fail-closed `PROVIDER_PENDING`; pytest factory stays stub. |
| 1.2 | 2026-09-12 | PROD-LIVE-015: SMTP `send_message` + Message-ID; SMS HTTP POST parses vendor `id`/`sid`; WhatsApp Graph parses `messages[0].id`. No invented live ids. |

---

## 1. Purpose (enterprise)

`p15_notification` is JeslotERP’s **omnichannel notification control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Output Management / Notification Framework** — templates, channels, status tracking  
- **Microsoft Dynamics 365** — customer journey / notification & email templates, preference centers  
- **Salesforce** — Email Templates, Notification Builder, Mobile Push, Messaging, unsubscribe  
- **Modern messaging platforms** — provider abstraction, webhooks, delivery receipts  

It is **not** `smtplib.sendmail` in a domain service. It is the system that makes ERP communications correct for:

1. **Multi-channel send** — Email, SMS, WhatsApp, Push, In-App, Webhook  
2. **Versioned templates** with **p06 localization** (ICU / locale packs)  
3. **Preference & consent** — opt-in/out, topics, quiet hours, DND  
4. **Routing & fallback** — email fails → in-app; SMS for OTP only, etc.  
5. **Durable delivery ledger** — queued → sent → delivered → bounced/failed  
6. **Provider adapters** — secrets via configuration; no keys in notification tables  
7. **Async dispatch** via **p14** jobs (never block HTTP on SMTP)  
8. **Digests & batching** — daily summary instead of spam  
9. **In-app inbox** — unread/read/archive for ERP shell  
10. **Attachments** via `media_id`; locale-aware subjects/bodies  

### Owns

| Domain | Examples |
|---|---|
| Channels & providers | email/sms/whatsapp/push/inapp/webhook |
| Templates & versions | keys, locales, merge fields |
| Topics / categories | security, ops, marketing, workflow |
| Preferences & consent | user/tenant opt rules |
| Notification requests | intent to notify |
| Deliveries | per-channel attempts |
| In-app inbox | user messages |
| Routing policies | fallback chains |
| Suppression | bounces, complaints, unsubscribes |
| Digests | schedules & buckets |
| Governance | publish templates, packs |

### Does **not** own

| Concern | Owner |
|---|---|
| Locale message catalog / ICU engine | `p06_localization` (templates reference keys or store ICU with locale) |
| Job execution workers | `p14_messaging` |
| Provider API secrets | `p03_configuration` |
| File bytes for attachments | `p08_file_media` |
| Human approval tasks | `p10_process` (emits notify intents) |
| Marketing campaign journey designer | Future; p15 is transactional + product notify core |

### Critical split: Notification vs Localization vs Messaging

| | **Notification (p15)** | **Localization (p06)** | **Messaging (p14)** |
|---|---|---|---|
| Owns | Who/when/which channel + delivery | How text is worded | Running the send job |
| Payload | Template key + data | Resolved strings | `notify.dispatch_channel` job |

**Rule:** Domain calls `notify.send` intent → p15 respects prefs → enqueues p14 → provider send → ledger update.

---

## 2. Architectural position

```text
Domain / Process / Event consumer
              │
              ▼
     notification request (intent)
              │
     prefs / consent / quiet hours / suppress
              │
     render template (p06 hydrate)
              │
     route channels + fallback
              │
     enqueue p14 jobs ──► provider adapters
              │
              ▼
     delivery ledger + in-app inbox + webhooks
```

**Hard rules**

1. Never send marketing/transactional without checking **suppression & consent**.  
2. OTP/security may use `priority=CRITICAL` + bypass quiet hours (policy-flagged topics only).  
3. No provider secrets in `notification` schema.  
4. No cross-schema FKs — UUID refs (`user_id`, `media_id`).  
5. RLS fail-closed on tenant notifications & inbox.  
6. Templates published immutably; edit via new version.

---

## 3. Advanced design principles

1. **Intent vs delivery** — request can fan out to many deliveries.  
2. **Topic taxonomy** — security, workflow, document, billing, system.  
3. **Template keys stable** — `notify.sales.order.cancelled.email`.  
4. **Locale resolution** — user → company → tenant → `en`.  
5. **Merge data schema** — declared facts; strict mode optional.  
6. **Channel capabilities** — SMS length, WhatsApp template pre-approval ids.  
7. **Provider health** — circuit break / failover provider.  
8. **Idempotent send** — `Idempotency-Key` per intent.  
9. **Quiet hours** — defer to `run_at` unless critical.  
10. **Digest bundling** — collapse eligible topics.  
11. **Bounce/complaint webhooks** — suppress future.  
12. **Unsubscribe tokens** — hashed storage.  
13. **In-app as channel** — first-class, not afterthought.  
14. **Attachment policy** — max size/count; virus-clean media only.  
15. **PII minimization** — don’t put secrets in SMS body.  
16. **Audit** — who triggered notify.  
17. **CQRS HTTP** — thin routers; dispatch services.  
18. **Packs** — seed ERP transactional templates.

---

## 4. Core concepts

### 4.1 Notification request (intent)

```text
request_id, topic_key, template_key,
recipients[], data{}, locale?,
channels[] | route_policy,
priority, tenant_id, company_id,
idempotency_key, scheduled_for?
```

### 4.2 Recipient

User id, employee, BP contact, raw address (email/phone) with type — raw addresses require permission & consent basis.

### 4.3 Template

- Channel-specific bodies (email html/text, sms, push title/body, inapp markdown)  
- `label_key` / ICU via p06 **or** embedded ICU per locale version  
- Merge fields allow-list  
- WhatsApp `provider_template_id` mapping  

### 4.4 Delivery lifecycle

```text
QUEUED → RENDERING → DISPATCHED → SENT → DELIVERED
                              └→ FAILED → (retry) → DEAD
                              └→ BOUNCED / COMPLAINED / SUPPRESSED / DEFERRED
```

### 4.5 Routing / fallback example

```text
1. push (if device)
2. inapp (always for users)
3. email (if opted)
4. sms (only if topic allows & phone verified)
```

### 4.6 Preference matrix

| Topic | Email | SMS | Push | InApp |
|---|---|---|---|---|
| security.otp | — | required | optional | optional |
| workflow.task | default on | off | on | on |
| marketing.* | opt-in | opt-in | opt-in | off |

---

## 5. Providers

| Channel | Example providers |
|---|---|
| Email | SMTP, SendGrid, SES, Graph |
| SMS | Twilio, MSG91, Gupshup |
| WhatsApp | Meta Cloud API, Gupshup |
| Push | FCM, APNs |
| Webhook | signed HTTP |

Each provider: `provider_key`, `secret_ref_key`, caps, sandbox flag.

---

## 6. Integration patterns

### 6.1 Process task assigned

p10 emits event → bridge → `notify.send` topic `workflow.task` template `notify.process.task_assigned`.

### 6.2 Document distributed

p09 distribute → notify recipients with controlled-copy link (signed URL from media/doc).

### 6.3 OTP

Identity calls notify with topic `security.otp`, channel SMS forced, short TTL template.

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `notify.template.read` | Read templates |
| `notify.template.manage` | Edit/publish templates |
| `notify.send` | Create send intents |
| `notify.send.critical` | Bypass quiet hours / security topics |
| `notify.prefs.manage` | Manage others’ prefs (admin) |
| `notify.inbox.read` | Read own inbox (default user) |
| `notify.delivery.read` | Inspect deliveries |
| `notify.provider.manage` | Providers |
| `notify.suppress.manage` | Suppressions |
| `notify.pack.install` | Packs |
| `notify.audit.read` | Audit |
| `notify.*` | Wildcard |

### RLS

FORCE RLS on tenant requests, deliveries, inbox, prefs.

---

## 8. Module layout

```text
platforms/p15_notification/
  application/
    services/
      send_orchestrator.py
      preference_guard.py
      quiet_hours.py
      template_renderer.py
      router.py
      digest.py
      suppression.py
      provider_failover.py
      inbox.py
    adapters/ email_*.py sms_*.py push_*.py whatsapp_*.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/ (enqueue) webhooks/
  tests/unit/prefs/ router/ render/
```

---

## 9. Domain events

| Event | When |
|---|---|
| `notify.request.accepted` / `suppressed` | Intent |
| `notify.delivery.sent` / `delivered` / `failed` / `bounced` | Channel |
| `notify.inbox.created` / `read` | In-app |
| `notify.template.activated` | Catalog |
| `notify.preference.changed` | Prefs |
| `notify.provider.unhealthy` | Ops |

Stream: register with p13 when Live.

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, topics, permissions |
| P2 | Templates + render + in-app |
| P3 | Email provider + ledger |
| P4 | Prefs/consent/quiet hours |
| P5 | SMS/push + routing fallback |
| P6 | Digests + suppression/bounces |
| P7 | WhatsApp template mapping |
| P8 | Packs + webhooks |
| P9 | Registry → **Live** |

---

## 11. Definition of Done (enterprise)

- [ ] Idempotent send returns same request  
- [ ] Quiet hours defer non-critical  
- [ ] Opt-out suppresses channel  
- [ ] Bounce adds suppression  
- [ ] Template activate immutable  
- [ ] p14 job used for provider I/O  
- [ ] Locale fallback via p06/user prefs  
- [ ] Tenant RLS on inbox/deliveries  
- [ ] No secrets in DB  
- [ ] No cross-schema FKs  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| SMTP from domain request thread | Enqueue notify job |
| Hardcoded English email HTML in services | Templates + i18n |
| Ignore unsubscribes | Suppression list |
| SMS long PII paragraphs | Short codes + in-app/email |
| Store provider API keys in notify tables | configuration secrets |
| Notify without topic | Always classify topic |

---

## 13. Related documents

- Schema: [`NOTIFICATION_SCHEMA.md`](NOTIFICATION_SCHEMA.md)  
- API: [`NOTIFICATION_API.md`](NOTIFICATION_API.md)  
- Localization: [`../06_localization/LOCALIZATION_GUIDE.md`](../06_localization/LOCALIZATION_GUIDE.md)  
- Messaging: [`../14_messaging/MESSAGING_GUIDE.md`](../14_messaging/MESSAGING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
