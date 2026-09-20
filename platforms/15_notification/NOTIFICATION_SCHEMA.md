# JeslotERP Notification Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** Enterprise schema contract (pre-implementation)  
**Package:** `platforms.p15_notification`  
**PostgreSQL schema:** `notification`  
**Companion:** [`NOTIFICATION_GUIDE.md`](NOTIFICATION_GUIDE.md) · [`NOTIFICATION_API.md`](NOTIFICATION_API.md)

> Runtime models: `platforms/p15_notification/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `notification` (never `p15`) |
| Tables | `ntf_*` |
| Template keys | `notify.{domain}.{event}.{channel?}` |
| Soft delete | Retire templates; retain deliveries |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant requests/inbox/prefs |
| Secrets | `secret_ref_key` only |

---

## 2. Complete table inventory (**62 tables**)

### 2.1 Topics, channels, providers (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `ntf_topic` | Topics (security, workflow, …) |
| 2 | `ntf_topic_policy` | Critical bypass, allowed channels |
| 3 | `ntf_channel` | EMAIL/SMS/WHATSAPP/PUSH/INAPP/WEBHOOK |
| 4 | `ntf_provider` | Provider registry |
| 5 | `ntf_provider_channel` | Provider↔channel |
| 6 | `ntf_provider_endpoint` | Endpoints / regions |
| 7 | `ntf_provider_health` | Health samples |
| 8 | `ntf_provider_failover` | Failover order |
| 9 | `ntf_feature_binding` | Feature gates |

### 2.2 Templates (9)

| # | Table | Purpose |
|---|---|---|
| 10 | `ntf_template` | Template catalog |
| 11 | `ntf_template_version` | Versions |
| 12 | `ntf_template_locale` | Per-locale content |
| 13 | `ntf_template_channel` | Channel variants |
| 14 | `ntf_merge_field` | Declared merge fields |
| 15 | `ntf_template_attachment` | Default media attachments |
| 16 | `ntf_whatsapp_mapping` | Provider template ids |
| 17 | `ntf_template_activation` | ACTIVE version |
| 18 | `ntf_template_preview` | Stored previews |

### 2.3 Recipients, prefs, consent (9)

| # | Table | Purpose |
|---|---|---|
| 19 | `ntf_contact_point` | Email/phone/device endpoints |
| 20 | `ntf_user_preference` | Per user topic/channel |
| 21 | `ntf_tenant_preference_default` | Tenant defaults |
| 22 | `ntf_consent_record` | Consent evidence |
| 23 | `ntf_quiet_hours` | Quiet hour windows |
| 24 | `ntf_dnd` | Do-not-disturb periods |
| 25 | `ntf_device_token` | Push device tokens |
| 26 | `ntf_unsubscribe_token` | Hashed unsub tokens |
| 27 | `ntf_preference_audit` | Pref change audit |

### 2.4 Requests & routing (7)

| # | Table | Purpose |
|---|---|---|
| 28 | `ntf_request` | Send intent |
| 29 | `ntf_request_recipient` | Recipients |
| 30 | `ntf_request_data` | Merge payload |
| 31 | `ntf_route_policy` | Routing/fallback |
| 32 | `ntf_route_step` | Ordered channel steps |
| 33 | `ntf_request_attachment` | media_ids |
| 34 | `ntf_idempotency` | Send idempotency |

### 2.5 Deliveries & provider I/O (8)

| # | Table | Purpose |
|---|---|---|
| 35 | `ntf_delivery` | Per-channel delivery |
| 36 | `ntf_delivery_attempt` | Attempts |
| 37 | `ntf_delivery_event` | Provider webhooks (open/click/bounce) |
| 38 | `ntf_render_snapshot` | Rendered body snapshot |
| 39 | `ntf_provider_message_ref` | External message ids |
| 40 | `ntf_dispatch_job_link` | p14 job id link |
| 41 | `ntf_throttle_counter` | Rate counters |
| 42 | `ntf_delivery_stats` | Aggregates |

### 2.6 In-app inbox (5)

| # | Table | Purpose |
|---|---|---|
| 43 | `ntf_inbox_message` | In-app messages |
| 44 | `ntf_inbox_state` | read/archived |
| 45 | `ntf_inbox_action` | CTA actions |
| 46 | `ntf_inbox_folder` | Optional folders |
| 47 | `ntf_inbox_subscription` | UI subscription prefs |

### 2.7 Suppression, digests, webhooks (8)

| # | Table | Purpose |
|---|---|---|
| 48 | `ntf_suppression` | Bounce/complaint/unsub |
| 49 | `ntf_suppression_reason` | Reasons |
| 50 | `ntf_digest_policy` | Digest rules |
| 51 | `ntf_digest_bucket` | Pending digest items |
| 52 | `ntf_digest_run` | Digest sends |
| 53 | `ntf_inbound_webhook` | Provider webhook inbox |
| 54 | `ntf_webhook_signature` | Sig validation meta |
| 55 | `ntf_complaint` | Spam complaints |

### 2.8 Governance & packs (7)

| # | Table | Purpose |
|---|---|---|
| 56 | `ntf_changeset` | Template changes |
| 57 | `ntf_approval` | Approvals |
| 58 | `ntf_package` | Template packs |
| 59 | `ntf_package_item` | Items |
| 60 | `ntf_catalog_audit` | Audit |
| 61 | `ntf_usage_stats` | Volume metrics |
| 62 | `ntf_simulation_run` | Dry-run renders |

**Plumbing:** `ntf_outbox`, `ntf_idempotency_key` (API)

**Implementation total with plumbing: 64 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `ntf_channel_code` | `EMAIL`, `SMS`, `WHATSAPP`, `PUSH`, `INAPP`, `WEBHOOK` |
| `ntf_request_status` | `ACCEPTED`, `DEFERRED`, `SUPPRESSED`, `ROUTING`, `COMPLETED`, `PARTIAL`, `FAILED`, `CANCELLED` |
| `ntf_delivery_status` | `QUEUED`, `RENDERING`, `DISPATCHED`, `SENT`, `DELIVERED`, `FAILED`, `BOUNCED`, `COMPLAINED`, `SUPPRESSED`, `DEFERRED`, `DEAD` |
| `ntf_priority` | `CRITICAL`, `HIGH`, `NORMAL`, `LOW` |
| `ntf_consent_basis` | `CONTRACT`, `LEGAL_OBLIGATION`, `CONSENT`, `LEGITIMATE_INTEREST` |
| `ntf_suppress_reason` | `HARD_BOUNCE`, `SOFT_BOUNCE`, `COMPLAINT`, `UNSUBSCRIBE`, `MANUAL`, `INVALID` |
| `ntf_template_lifecycle` | `DRAFT`, `PUBLISHED`, `ACTIVE`, `RETIRED` |
| `ntf_inbox_status` | `UNREAD`, `READ`, `ARCHIVED`, `DELETED` |

---

## 4. Topics & providers

### 4.1 `ntf_topic`

| Column | Type | Notes |
|---|---|---|
| `topic_key` | VARCHAR(100) UNIQUE | `security.otp`, `workflow.task` |
| `name` | VARCHAR(150) | |
| `is_critical` | BOOLEAN | Quiet-hour bypass eligible |
| `requires_consent` | BOOLEAN | |
| `default_priority` | VARCHAR(20) | |
| `label_key` | VARCHAR(200) NULL | |

### 4.2 `ntf_topic_policy`

| Column | Type | Notes |
|---|---|---|
| `topic_id` | UUID | |
| `allowed_channels` | JSONB | |
| `forced_channels` | JSONB NULL | e.g. SMS for OTP |
| `allow_digest` | BOOLEAN | |
| `bypass_quiet_hours` | BOOLEAN | |

### 4.3 `ntf_provider`

| Column | Type | Notes |
|---|---|---|
| `provider_key` | VARCHAR(50) UNIQUE | `ses_primary` |
| `channel_code` | VARCHAR(20) | |
| `secret_ref_key` | VARCHAR(150) | |
| `is_active` | BOOLEAN | |
| `is_sandbox` | BOOLEAN | |
| `config` | JSONB | non-secret |

---

## 5. Templates

### 5.1 `ntf_template`

| Column | Type | Notes |
|---|---|---|
| `template_key` | VARCHAR(150) UNIQUE | |
| `topic_id` | UUID | |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `strict_merge` | BOOLEAN | |
| `is_active` | BOOLEAN | |

### 5.2 `ntf_template_version`

| Column | Type | Notes |
|---|---|---|
| `template_id` | UUID | |
| `version_number` | INT | |
| `lifecycle` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |
| `activated_at` | TIMESTAMPTZ NULL | |

### 5.3 `ntf_template_locale` / channel

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `locale_code` | VARCHAR(35) | |
| `channel_code` | VARCHAR(20) | |
| `subject_icu` | TEXT NULL | Email/push |
| `body_icu` | TEXT | |
| `body_html_icu` | TEXT NULL | Email |
| `i18n_key_prefix` | VARCHAR(200) NULL | Optional hydrate via p06 |
| `whatsapp_mapping_id` | UUID NULL | |

### 5.4 `ntf_merge_field`

| Column | Type | Notes |
|---|---|---|
| `template_id` | UUID | |
| `field_key` | VARCHAR(80) | |
| `data_type` | VARCHAR(20) | |
| `required` | BOOLEAN | |
| `example` | JSONB NULL | |

---

## 6. Preferences & contacts

### 6.1 `ntf_contact_point`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID NULL | |
| `party_id` | UUID NULL | BP ref |
| `channel_code` | VARCHAR(20) | |
| `address` | VARCHAR(320) | email/e164 |
| `is_verified` | BOOLEAN | |
| `is_primary` | BOOLEAN | |

### 6.2 `ntf_user_preference`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID | |
| `topic_id` | UUID | |
| `channel_code` | VARCHAR(20) | |
| `is_enabled` | BOOLEAN | |
| `digest_mode` | VARCHAR(20) NULL | INSTANT/DAILY/WEEKLY |

**Unique:** `(tenant_id, user_id, topic_id, channel_code)`.

### 6.3 `ntf_quiet_hours`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` / `user_id` | | Scope |
| `timezone` | VARCHAR(100) | |
| `start_minute` | INT | 0–1439 |
| `end_minute` | INT | |
| `days_of_week` | JSONB | |

---

## 7. Requests & deliveries

### 7.1 `ntf_request`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | |
| `company_id` | UUID NULL | |
| `topic_id` | UUID | |
| `template_id` | UUID | |
| `template_version_id` | UUID NULL | Pin |
| `status` | VARCHAR(20) | |
| `priority` | VARCHAR(20) | |
| `locale_code` | VARCHAR(35) NULL | |
| `idempotency_key` | VARCHAR(120) NULL | |
| `scheduled_for` | TIMESTAMPTZ NULL | |
| `created_by` | UUID NULL | |
| `correlation_id` | UUID NULL | |
| `suppression_reason` | VARCHAR(30) NULL | |

### 7.2 `ntf_request_recipient`

| Column | Type | Notes |
|---|---|---|
| `request_id` | UUID | |
| `user_id` | UUID NULL | |
| `contact_point_id` | UUID NULL | |
| `address_override` | VARCHAR(320) NULL | Rare |
| `display_name` | VARCHAR(150) NULL | |

### 7.3 `ntf_delivery`

| Column | Type | Notes |
|---|---|---|
| `request_id` | UUID | |
| `recipient_id` | UUID | |
| `channel_code` | VARCHAR(20) | |
| `provider_id` | UUID NULL | |
| `status` | VARCHAR(20) | |
| `attempt_count` | INT | |
| `next_attempt_at` | TIMESTAMPTZ NULL | |
| `provider_message_id` | VARCHAR(200) NULL | |
| `error_code` | VARCHAR(50) NULL | |
| `sent_at` / `delivered_at` | TIMESTAMPTZ NULL | |

### 7.4 `ntf_render_snapshot`

Stores rendered subject/body (redaction policy for PII topics); used for audit & debug.

---

## 8. Inbox

### 8.1 `ntf_inbox_message`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID | |
| `request_id` | UUID NULL | |
| `delivery_id` | UUID NULL | |
| `topic_id` | UUID | |
| `title` | VARCHAR(200) | |
| `body` | TEXT | |
| `status` | VARCHAR(20) | |
| `action_url` | TEXT NULL | |
| `created_at` | TIMESTAMPTZ | |
| `read_at` | TIMESTAMPTZ NULL | |

---

## 9. Suppression & digests

### 9.1 `ntf_suppression`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | Null = global address |
| `channel_code` | VARCHAR(20) | |
| `address` | VARCHAR(320) | |
| `reason` | VARCHAR(30) | |
| `expires_at` | TIMESTAMPTZ NULL | Soft bounce TTL |
| `is_active` | BOOLEAN | |

### 9.2 `ntf_digest_bucket`

Pending items awaiting digest run; flushed by scheduler job.

---

## 10. Governance & packs

- Template changesets/approvals before activate in production  
- Packages: `notify.security@1.0.0`, `notify.workflow@1.0.0`, `notify.document@1.0.0`  
- Simulation runs for render dry-run  

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `ntf_outbox` | Domain events |
| `ntf_idempotency_key` | API idempotency |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| Topics/templates system | Read auth; manage permission |
| Requests, deliveries, inbox, prefs | FORCE `tenant_id` |
| Suppressions | Tenant or global with admin |
| Providers | Admin only |

---

## 13. Seed minimum

1. Channels list  
2. Topics: `security.otp`, `security.login`, `workflow.task`, `document.released`, `system.alert`  
3. Route policies default  
4. In-app + email templates for workflow.task  
5. Quiet hours default off  
6. Handler bridge `notify.dispatch_channel` in messaging pack  
7. Permissions `notify.*`  

---

## 14. ER overview

```text
topic ── policy
template ── versions ── locale/channel content / merge_fields
provider ── channel / health / failover

request ── recipients / data / attachments
       ── deliveries ── attempts / events / render_snapshot
       ── inbox_message

prefs / consent / quiet_hours / device_tokens
suppression / digest_bucket
packages
```

---

## 15. Implementation notes

1. Dispatch always via p14 job `notify.dispatch_channel`.  
2. Render uses p06 when `i18n_key_prefix` set; else locale ICU on template.  
3. Webhook endpoints verify signatures before mutating delivery.  
4. Hard bounce → suppression without expiry; soft bounce with TTL.  
5. Split models: `catalog`, `template`, `prefs`, `request`, `delivery`, `inbox`, `suppress`, `governance`, `plumbing`.
