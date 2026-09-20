# Event Bus Platform — Requirements Traceability Matrix (RTM)

**Platform:** `p13_event_bus` · **Schema:** `event_bus`  
**Sources:** EVENT_BUS_GUIDE.md · EVENT_BUS_SCHEMA.md · EVENT_BUS_API.md · docs/sample.md  
**Verification date:** 2026-09-12  
**Test command:** `pytest platforms/p13_event_bus/tests -q --tb=short` → **39 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| EB-G-01 | GUIDE §1 | Domain event control plane & outbox relay | `platforms/p13_event_bus` ModulePlugin | Done | `test_eb_module_*` | Passed |
| EB-G-02 | GUIDE §1 | CloudEvents 1.0 + JeslotERP extensions | publish/relay envelope builder | Done | publish/pull API + engine tests | Passed |
| EB-G-03 | GUIDE §3 | Schema registry + compatibility | `activate_schema` + `_compat_ok` | Done | `test_eb_schema_*`, `test_eb_compat_*`, API activate | Passed |
| EB-G-04 | GUIDE §3 | Topics + type bindings | topics store + APIs | Done | `test_eb_topics_*` | Passed |
| EB-G-05 | GUIDE §3 | Subscriptions + filters | subscriptions + `filter_engine` | Done | filter unit + subscription API | Passed |
| EB-G-06 | GUIDE §3 | Outbox source registry + relay cursor | outbox sources + `relay_tick` | Done | `test_eb_relay_*` | Passed |
| EB-G-07 | GUIDE §3 | Idempotent outbox_row → event_id | `outbox_event_map` | Done | relay mapping tests | Passed |
| EB-G-08 | GUIDE §3 | At-least-once pull + visibility lease | `pull` / ack / nack | Done | `test_eb_publish_success_and_pull_ack` | Passed |
| EB-G-09 | GUIDE §3 | Retry/backoff → DLQ + redrive | nack + redrive | Done | `test_eb_nack_to_dlq_and_redrive` | Passed |
| EB-G-10 | GUIDE §3 | Poison classification | poison list on relay fail | Done | relay tick failure path | Passed (unit via relay) |
| EB-G-11 | GUIDE §3 | Replay scoped + auth | replay APIs + `event.replay` | Done | `test_eb_replay_scoped` | Passed |
| EB-G-12 | GUIDE §3 | Consumer idempotency | consumer-groups idempotency API | Done | `test_eb_consumer_idempotency_helper` | Passed |
| EB-G-13 | GUIDE §3 | TENANT_REQUIRED for business types | publish validation | Done | `test_eb_publish_requires_tenant` | Passed |
| EB-G-14 | GUIDE §3 | Payload size cap | `PAYLOAD_MAX_BYTES` | Done | `test_eb_payload_too_large` | Passed |
| EB-G-15 | GUIDE §3 | Ordering KEY opt-in | `_ordering_blocked` | Done | `test_eb_ordering_key_*` | Passed |
| EB-G-16 | GUIDE §7 | Permissions `event.*` | permissions catalog + gates | Done | `test_eb_permission_denied_*` | Passed |
| EB-G-17 | GUIDE §7 | RLS FORCE tenant tables | Alembic `f13b1c2d3e4f` | Done | migration applied | Passed (DB) |
| EB-G-18 | GUIDE §9 | Meta outbox `jesloterp:event_bus:outbox` | `OUTBOX_STREAM` + `eb_outbox` | Done | health reports stream | Passed |
| EB-SOR-01 | TASK-SOR-015 | HTTP publish event+outbox same commit | `persist_published_event` + `/publish` | Done | `test_persist_published_event_writes_event_and_outbox_same_commit` | Passed |
| EB-G-19 | GUIDE §14 | Seed types/topics/subscription/policies | `seed_defaults` + Alembic seeds | Done | catalog list seeded types | Passed |
| EB-S-01 | SCHEMA §2 | 61 domain + 3 plumbing = 64 tables | ORM models | Done | `test_eb_module_tables_count_64` + DB 64 | Passed |
| EB-S-02 | SCHEMA §1 | Schema name `event_bus` never p13 | `EVENT_BUS_SCHEMA` | Done | module/tables tests | Passed |
| EB-S-03 | SCHEMA §1 | No cross-schema FKs | UUID refs; outbox sources by name | Done | ORM review | Passed |
| EB-S-04 | SCHEMA §14 | Seed domains/contract/retry/permissions | store + Alembic | Done | contract/health/package tests | Passed |
| EB-A-01 | API §4 | Error codes EB_* | `domain/exceptions.py` + handlers | Done | 403/409/413/422 API tests | Passed |
| EB-A-02 | API §5 | Permission codes | `EVENT_BUS_PERMISSIONS` + migration | Done | permission denied test | Passed |
| EB-A-03 | API §6 | Types/schemas CRUD + activate/validate | catalog router | Done | catalog/schema API tests | Passed |
| EB-A-04 | API §7 | Topics/subscriptions/consumer-groups | topology router | Done | topics/subscription tests | Passed |
| EB-A-05 | API §8 | Admin publish + Idempotency-Key | publish router | Done | publish success/tenant/large | Passed |
| EB-A-06 | API §9 | Outbox sources + lag + batches | publish_relay router | Done | relay API tests | Passed |
| EB-A-07 | API §10 | Pull/ack/nack + idempotency helper | delivery router | Done | pull/ack/idempotency tests | Passed |
| EB-A-08 | API §11 | Dispatch tick + worker result | internal router | Done | `test_eb_internal_*` | Passed |
| EB-A-09 | API §12 | Inspect events/deliveries/attempts | delivery router | Done | covered in pull/publish flows | Passed |
| EB-A-10 | API §13 | DLQ list/redrive/discard | delivery router | Done | dlq/redrive test | Passed |
| EB-A-11 | API §14 | Replay start/get/cancel | delivery router | Done | replay test | Passed |
| EB-A-12 | API §15 | Grants/retry/packages/changesets | governance router | Done | grants/packages tests | Passed |
| EB-A-13 | API §16 | Outbox contract + validate-sample | publish_relay | Done | `test_eb_outbox_contract_*` | Passed |
| EB-A-14 | API §17 | Health + metrics lag/dlq | governance + internal | Done | health test | Passed |
| EB-A-15 | API internal | publish/relay/tick/register-handler | internal router | Done | internal + relay tests | Passed |
| EB-W-01 | Wiring | main.py EventBusModule + handlers | `apps/api/main.py` | Done | `test_eb_app_loads_p13_after_p12` | Passed |
| EB-W-02 | Wiring | alembic env import models | `alembic/env.py` | Done | upgrade applied | Passed |
| EB-W-03 | Wiring | Migrations after feature head | `f13a0b1c2d3e` → `f13b1c2d3e4f` | Done | `alembic current` | Passed |
| EB-T-01 | sample.md | ≥2 variations per major area | api/engine/filter/schema/module tests | Done | 31 tests | Passed |

**Coverage note:** Runtime catalog/relay/delivery uses in-memory `EventBusCatalogStore` (same pattern as p11/p12). ORM models cover all 64 tables for Alembic `create_all`. Not a durable Kafka/broker — in-process store + Postgres schema is intentional for this platform phase.
