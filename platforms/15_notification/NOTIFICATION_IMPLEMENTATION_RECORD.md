# Notification Platform — Implementation Record

**Platform:** `p15_notification`  
**Date:** 2026-09-11  
**Scope:** Backend only (per `docs/tasks/task_p15_notification.md`)  
**Verification:** `pytest platforms/p15_notification/tests -q --tb=line` → **43 passed**; ORM `notification` table count → **64**; module load order p15 after p14 → **passed**

---

## 1. Overview & Objective

Implement JeslotERP notification platform end-to-end: schema `notification`, 64 `ntf_*` tables (62 domain + outbox + idempotency_key), ModulePlugin `p15_notification` (depends on `p01_identity`, `p06_localization`, `p14_messaging`), send orchestration with prefs/quiet hours/suppression, in-app inbox, templates, providers, digests, webhooks, packs, permissions, RLS migrations, tests, RTM, and status updates.

## 2. All 3 Source Documents Reviewed

| Document | Path | Role |
| --- | --- | --- |
| GUIDE | `docs/platforms/15_notification/NOTIFICATION_GUIDE.md` | Architecture, channels, prefs, DoD |
| SCHEMA | `docs/platforms/15_notification/NOTIFICATION_SCHEMA.md` | 62 + 2 plumbing tables, enums, seed |
| API | `docs/platforms/15_notification/NOTIFICATION_API.md` | Public/internal HTTP surface, errors, permissions |

Also followed `docs/tasks/task_p15_notification.md` and mirrored `platforms/p14_messaging/`.

## 3. Existing Backend Architecture Reviewed

- ModulePlugin registration and topo-sort deps in `apps/api/main.py`
- Alembic `env.py` dynamic model imports
- p14 patterns: in-memory catalog store, thin routers, exception handlers, permission deps, outbox stream, dual migrations (schema + RLS)
- Shared `EnterpriseBase` / `PlatformBase` / catalog base ORM bases
- No cross-schema FKs — UUID refs only (`user_id`, `media_id`, `job_id`)

## 4. Requirements Identified

See `NOTIFICATION_RTM.md` (100% mapped). Major themes: idempotent send, critical send permission, quiet hours deferral, preference opt-out, suppression/bounce, template ACTIVE pin + immutability, preview dry-run, inbox CRUD, provider secret_ref_key only, p14 dispatch job links, digests, webhooks signature verification, packs, 64 ORM tables, wiring + docs.

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | NTF_* codes + channel/lifecycle enums | `domain/` | API §4 / SCHEMA §3 | exception handler + API status tests |
| Catalog store | In-memory send/prefs/inbox/templates | `application/services/catalog_store.py` | Runtime like MessagingCatalogStore | 43 pytest |
| Service facades | send/prefs/quiet/render/router/digest/suppress/failover/inbox | `application/services/*.py` | GUIDE §8 layout | unit tests |
| Provider stubs | email/sms/push/whatsapp no-op adapters | `application/adapters/` | GUIDE providers; no SMTP in HTTP | dispatch marks SENT |
| ORM 64 tables | Split model modules | `infrastructure/persistence/models/` | Alembic create_all | count test |
| HTTP APIs | Public + internal routers | `infrastructure/http/` | NOTIFICATION_API | contract tests |
| Permissions | `notify.*` catalog | `application/permissions/` | API §5 | gate test + migration seed |
| Module | `NotificationModule` deps p01+p06+p14 | `infrastructure/module.py` | registry | load-order test |
| Migrations | schema + RLS | `alembic/versions/f15*.py` | Live DB path | revision chain |
| Wiring | main + env.py | `apps/api/main.py`, `alembic/env.py` | mandatory | module load test |

## 6. Files/Modules/Services Created or Modified

**Created (high level):**
- `platforms/p15_notification/**` (domain, application, infrastructure, tests)
- `alembic/versions/f15a0b1c2d3e_create_notification_schema.py`
- `alembic/versions/f15b1c2d3e4f_enable_notification_rls.py`
- `docs/platforms/15_notification/NOTIFICATION_RTM.md`
- `docs/platforms/15_notification/NOTIFICATION_IMPLEMENTATION_RECORD.md` (this file)

**Modified:**
- `apps/api/main.py` — NotificationModule after MessagingModule + exception handlers
- `alembic/env.py` — import p15 models
- `IMPLEMENTATION_TASKS.md` — TASK-001 marked complete
- `IMPLEMENTATION_STATUS.md` — advanced to TASK-002

**Not modified (per brief):** GUIDE / SCHEMA / API requirement docs; task brief.

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `f15a0b1c2d3e` | CREATE SCHEMA `notification`; create_all 64 tables; seed channels, topics+policies, default.user route, workflow.task template ACTIVE, suppression reasons, `notify.*` permissions + admin grants | Created (apply via alembic upgrade) |
| `f15b1c2d3e4f` | ENABLE + FORCE RLS on tenant-scoped notification tables | Created |

**Down revision chain:** `f14b1c2d3e4f` → `f15a0b1c2d3e` → `f15b1c2d3e4f`  
**Note:** Unique `f15*` IDs follow p14 migration ID style.

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/notifications`:
- Send: `/send`, `/send-critical`, `/preview`, requests get/deliveries/cancel, deliveries get/events
- Inbox: list, unread-count, get, read, read-all, archive, delete
- Prefs: me get/put, quiet-hours me, admin user prefs, tenant defaults, devices, unsubscribe
- Templates: list/create/get/versions/create version, locale put, publish/activate/simulate
- Catalog: topics, route-policies, providers (+ test/health)
- Ops: suppressions, digests policies, webhooks, packages, changesets/approvals, stats, audit

Internal `/internal/v1/notifications`:
- `/dispatch/{delivery_id}`, `/send`, `/digests/flush`, `/webhooks/{provider_key}`

## 9. Business Rules & Workflows Implemented

1. Send accepts intent; resolves channels via route policy or forced topic channels.
2. Preference opt-out → delivery SUPPRESSED; quiet hours → DEFERRED (non-bypass topics).
3. Suppression list blocks matching address/channel.
4. Critical path requires `notify.send.critical` (HTTP send-critical + orchestrator flag).
5. ACTIVE template version pinned at accept; ACTIVE locale content immutable.
6. Preview/simulate render without writing request ledger.
7. Dispatch creates conceptual p14 job link (`notify.dispatch_channel`); stub adapter marks SENT.
8. INAPP channel creates inbox message for user recipients.
9. Webhook HMAC signature verified; hard bounce creates non-expiring suppression.
10. Providers expose `secret_ref_key` only — never raw API keys.

## 10. Validation, Permissions & Error Handling

- `require_notify_permission` + `notify.*` wildcard; platform admin roles bypass.
- Domain exceptions mapped via `register_notification_exception_handlers` to StandardResponse error envelope with NTF_* codes.
- Idempotency conflict → 409; webhook bad signature → 403; missing entities → 404; validation → 422.

## 11. Integrations Implemented

| Integration | How |
| --- | --- |
| p14 messaging | `ntf_dispatch_job_link` + handler key `notify.dispatch_channel` (seeded in p14 messaging migration) |
| p01 identity | JWT auth via metadata CurrentUser; permission seed into identity when present |
| p06 localization | Template `i18n_key_prefix` column + ICU merge render; locale fallback en |
| p03 configuration | Providers store `secret_ref_key` only |
| Provider I/O | Stub adapters (email/sms/push/whatsapp) — no network SMTP/SMS |

## 12. Test Cases Created for Each Functionality

| Group | Tests (examples) | ≥2 variations |
| --- | --- | --- |
| Module | table count 64, deps, load order, outbox stream | Yes |
| Send API | idempotent, conflict, critical denied/ok, preview, cancel | Yes |
| Inbox | list/unread, read/archive/delete | Yes |
| Prefs/devices/quiet | opt-out suppress, devices CRUD, quiet defer | Yes |
| Templates | lifecycle publish/activate, ACTIVE immutable | Yes |
| Topics/routes/providers | create topic, secret_ref, route steps | Yes |
| Suppress/digest/webhook | suppress blocks, flush, invalid sig + bounce | Yes |
| Packages/audit/internal | checksum mismatch, stats, dispatch SENT, internal send | Yes |
| Unit services | prefs, quiet, render, router, suppression, send orch | Yes |

## 13. Test Execution Results

```text
python -m pytest platforms/p15_notification/tests -q --tb=line
43 passed, 4 warnings in ~8.4s
```

Smoke: `load_modules()` includes `p15_notification` after `p14_messaging` — passed.

## 14. Requirements Traceability Matrix (RTM)

Full matrix: [`NOTIFICATION_RTM.md`](NOTIFICATION_RTM.md) — 100% requirement coverage with test status Passed.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Prefs unit test accidentally nulled method | Removed bad line; call `put_prefs` correctly |
| Suppression unit used tenant-scoped row without tenant filter | Pass matching `tenant_id` into `is_suppressed` |
| Webhook body + Request dual params | Use raw `request.body()` only |
| ProviderFailover initially written into inbox.py | Split into `provider_failover.py` + proper `inbox.py` |

## 16. Regression/Existing Functionality Verification

- Notification suite isolated; load_modules smoke asserts p14 still present and p15 after it.
- Did not re-run full monorepo suite in this task (focused validation per brief). Prior platforms unchanged except main/env wire-in.

## 17. Final Coverage & Completion Status

| Item | Status |
| --- | --- |
| 64 ORM tables | Done |
| All API groups in NOTIFICATION_API | Done |
| Services/adapters layout | Done |
| Alembic schema + RLS | Done |
| main.py + env.py wire-in | Done |
| ≥2 tests per group | Done (43 passed) |
| RTM 100% | Done |
| Implementation record (18 sections) | Done |
| TASK-001 status files | Done |

## 18. Remaining Issues or Limitations

1. **In-memory runtime store** — production durable Postgres repositories not yet swapped behind the catalog store (same phase pattern as p14).
2. **Provider ports:** SMTP/SMS/WhatsApp live adapters exist and return `PROVIDER_PENDING` until a real vendor client + credentials are present. Pytest factory stays stub. No fake live message ids.
3. **Alembic upgrade not executed against live DB in this session** — migrations authored and importable; apply with `alembic upgrade head` in target environments.
4. **p06 hydrate** — ICU templates rendered locally; full p06 catalog hydrate path is optional when `i18n_key_prefix` set (column present).
5. **Media attachment AVAILABLE checks** — attachment media_ids accepted on send body; deep p08 quarantine validation not wired (UUID ref only).
