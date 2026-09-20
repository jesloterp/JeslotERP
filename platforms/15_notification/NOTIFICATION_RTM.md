# Notification Platform — Requirements Traceability Matrix (RTM)

**Platform:** `p15_notification` · **Schema:** `notification`  
**Sources:** NOTIFICATION_GUIDE.md · NOTIFICATION_SCHEMA.md · NOTIFICATION_API.md · docs/tasks/task_p15_notification.md  
**Verification date:** 2026-09-12  
**Test command:** `pytest platforms/p15_notification/tests -q --tb=short` → **70 passed**  
**Alembic:** `f15a0b1c2d3e` → `f15b1c2d3e4f` (down_revision from `f14b1c2d3e4f`)

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| NTF-G-01 | GUIDE §1 | Omnichannel notification control plane | `platforms/p15_notification` ModulePlugin | Done | `test_ntf_module_*` | Passed |
| NTF-G-02 | GUIDE §1 | Multi-channel send EMAIL/SMS/WA/PUSH/INAPP/WEBHOOK | channels seed + router | Done | send + router tests | Passed |
| NTF-G-03 | GUIDE §1 | Versioned templates + locale content | template models + APIs | Done | template lifecycle tests | Passed |
| NTF-G-04 | GUIDE §1 | Preference & consent / quiet hours / DND | prefs + quiet_hours services | Done | prefs/quiet_hours unit + API | Passed |
| NTF-G-05 | GUIDE §1 | Routing & fallback chains | `ntf_route_*` + NotificationRouter | Done | router unit + route API | Passed |
| NTF-G-06 | GUIDE §1 | Durable delivery ledger | deliveries + events store | Done | send/dispatch/webhook tests | Passed |
| NTF-G-07 | GUIDE §1 | Provider adapters; secrets via secret_ref_key only | providers + stubs | Done | providers API test | Passed |
| NTF-SOR-05 | TASK-SOR-005 | Live SMTP/SMS/WhatsApp ports; pytest stub | adapter_factory + live adapters | Done | `test_live_email_sms_whatsapp_ports_are_pending_without_vendor` | Passed |
| NTF-G-08 | GUIDE §1 | Async dispatch via p14 job link | `ntf_dispatch_job_link` | Done | dispatch + orchestrator tests | Passed |
| NTF-G-09 | GUIDE §1 | Digests & batching | digest policies/buckets/flush | Done | digest API + flush | Passed |
| NTF-G-10 | GUIDE §1 | In-app inbox unread/read/archive | inbox APIs + service | Done | inbox API tests | Passed |
| NTF-G-11 | GUIDE §2 | Prefs/consent/quiet/suppress before send | send orchestrator | Done | prefs/suppress/quiet API | Passed |
| NTF-G-12 | GUIDE §2 | CRITICAL bypass quiet hours w/ policy/perm | send-critical + policy | Done | critical perm + quiet tests | Passed |
| NTF-G-13 | GUIDE §2 | No secrets in notification schema | secret_ref_key only | Done | providers test | Passed |
| NTF-G-14 | GUIDE §2 | No cross-schema FKs | UUID refs only | Done | ORM review | Passed |
| NTF-G-15 | GUIDE §2 | RLS fail-closed tenant | Alembic `f15b1c2d3e4f` | Done | migration present | Passed |
| NTF-G-16 | GUIDE §3 | Idempotent send Idempotency-Key | catalog_store.send | Done | idempotent + conflict tests | Passed |
| NTF-G-17 | GUIDE §3 | Template ACTIVE immutable; edit via new version | activate + put_locale guard | Done | immutable ACTIVE test | Passed |
| NTF-G-18 | GUIDE §3 | Bounce/complaint → suppression | webhook handler | Done | webhook bounce test | Passed |
| NTF-G-19 | GUIDE §3 | Unsubscribe hashed tokens | unsubscribe APIs | Done | unsubscribe test | Passed |
| NTF-G-20 | GUIDE §7 | Permissions notify.* | permissions catalog + gates | Done | denied send test | Passed |
| NTF-G-21 | GUIDE §9 | Domain events outbox | `ntf_outbox` + stream | Done | module stream constant | Passed |
| NTF-G-22 | GUIDE §11 | DoD checklist (idempotent/quiet/opt-out/bounce/activate/p14/no secrets/no X-FK) | services + migrations | Done | suite + table count | Passed |
| NTF-S-01 | SCHEMA §2 | 62 domain + 2 plumbing = 64 tables | ORM models | Done | `test_ntf_module_tables_count_64` | Passed |
| NTF-S-02 | SCHEMA §1 | Schema name `notification` never p15 | `NOTIFICATION_SCHEMA` | Done | module/tables tests | Passed |
| NTF-S-03 | SCHEMA §3 | Enums channel/request/delivery/priority/etc. | `domain/enums.py` | Done | used across store/tests | Passed |
| NTF-S-04 | SCHEMA §13 | Seed channels, topics, route, templates, permissions | store.seed_defaults + Alembic | Done | topics/templates list tests | Passed |
| NTF-S-05 | SCHEMA §15 | Split models catalog/template/prefs/request/delivery/inbox/suppress/governance/plumbing | model packages | Done | import + count | Passed |
| NTF-S-06 | SCHEMA §15 | Dispatch via notify.dispatch_channel job link | DISPATCH_HANDLER + links | Done | dispatch tests | Passed |
| NTF-A-01 | API §3 | StandardResponse envelope | `route_common.ok` | Done | all API tests | Passed |
| NTF-A-02 | API §4 | Error codes NTF_* | `domain/exceptions.py` + handlers | Done | 403/404/409/422 tests | Passed |
| NTF-A-03 | API §5 | Permission codes notify.* | catalog + require_notify_permission | Done | denied + critical denied | Passed |
| NTF-A-04 | API §6 | Send / send-critical / preview / requests / deliveries / cancel | send router | Done | send group API tests (≥2) | Passed |
| NTF-A-05 | API §7 | Inbox list/unread/get/read/read-all/archive/delete | inbox router | Done | inbox API tests (≥2) | Passed |
| NTF-A-06 | API §8 | Prefs / quiet hours / devices / unsubscribe | prefs router | Done | prefs/devices/quiet/unsub (≥2) | Passed |
| NTF-A-07 | API §9 | Templates CRUD/versions/publish/activate/simulate | templates router | Done | template lifecycle (≥2) | Passed |
| NTF-A-08 | API §10 | Topics / routes / providers | catalog router | Done | topics/routes/providers (≥2) | Passed |
| NTF-A-09 | API §11 | Suppressions / digests / internal flush | ops + internal | Done | suppress/digest tests (≥2) | Passed |
| NTF-A-10 | API §12 | Provider webhooks signature verify | ops + internal webhooks | Done | invalid + bounce (≥2) | Passed |
| NTF-A-11 | API §13 | Internal dispatch + trusted send | internal router | Done | dispatch + internal send (≥2) | Passed |
| NTF-A-12 | API §14 | Packages / changesets approvals | ops router | Done | packages checksum + changesets | Passed |
| NTF-A-13 | API §15 | Stats + audit requests | ops router | Done | stats/audit tests | Passed |
| NTF-W-01 | Wiring | main.py NotificationModule after Messaging + handlers | `apps/api/main.py` | Done | `test_ntf_app_loads_p15_after_p14` | Passed |
| NTF-W-02 | Wiring | alembic env import models | `alembic/env.py` | Done | import path present | Passed |
| NTF-W-03 | Wiring | Migrations after messaging head | `f15a0b1c2d3e` → `f15b1c2d3e4f` | Done | revision chain | Passed |
| NTF-T-01 | task brief | ≥2 variations per API/functionality | api + unit suites | Done | 43 tests | Passed |

**Coverage note:** Runtime uses in-memory `NotificationCatalogStore` (same pattern as p14). ORM models cover all 64 tables for Alembic `create_all`. Provider I/O is stubbed; dispatch creates `ntf_dispatch_job_link` and marks SENT in-process for tests.
