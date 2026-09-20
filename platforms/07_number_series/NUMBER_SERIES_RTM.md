# JeslotERP Number Series Platform (p07) — Requirements Traceability Matrix

**Date:** 2026-09-10  
**Package:** `platforms.p07_number_series`  
**PostgreSQL schema:** `number_series`  
**Sources:** `NUMBER_SERIES_GUIDE.md`, `NUMBER_SERIES_SCHEMA.md`, `NUMBER_SERIES_API.md`, `docs/sample.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| NS-SCH-001 | SCHEMA §2.1 | `ns_document_type` | `models/catalog.py` | Done | `test_all_67_number_series_tables_registered` | Pass |
| NS-SCH-002 | SCHEMA §2.1 | `ns_series_object` | `models/catalog.py` + seed | Done | module + catalog HTTP | Pass |
| NS-SCH-003 | SCHEMA §2.1 | `ns_series_object_alias` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-004 | SCHEMA §2.1 | `ns_document_binding` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-005 | SCHEMA §2.1 | `ns_series_group` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-006 | SCHEMA §2.1 | `ns_series_group_member` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-007 | SCHEMA §2.1 | `ns_channel` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-008 | SCHEMA §2.1 | `ns_feature_binding` | `models/catalog.py` | Done | table registry | Pass |
| NS-SCH-009 | SCHEMA §2.2 | `ns_series_definition` | `models/definition.py` | Done | table + catalog API | Pass |
| NS-SCH-010 | SCHEMA §2.2 | `ns_segment_type` | `models/definition.py` + seed | Done | seeds test | Pass |
| NS-SCH-011 | SCHEMA §2.2 | `ns_series_segment` | `models/definition.py` | Done | segments API | Pass |
| NS-SCH-012 | SCHEMA §2.2 | `ns_format_pattern` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-013 | SCHEMA §2.2 | `ns_check_digit_rule` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-014 | SCHEMA §2.2 | `ns_charset_rule` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-015 | SCHEMA §2.2 | `ns_validation_rule` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-016 | SCHEMA §2.2 | `ns_series_template` | `models/definition.py` + seed | Done | templates API | Pass |
| NS-SCH-017 | SCHEMA §2.2 | `ns_series_template_item` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-018 | SCHEMA §2.2 | `ns_definition_activation` | `models/definition.py` | Done | table registry | Pass |
| NS-SCH-019 | SCHEMA §2.3 | `ns_scope_dimension` | `models/scope.py` + seed | Done | seeds test | Pass |
| NS-SCH-020 | SCHEMA §2.3 | `ns_series_scope` | `models/scope.py` | Done | table registry | Pass |
| NS-SCH-021 | SCHEMA §2.3 | `ns_series_assignment` | `models/scope.py` + store | Done | assignments API | Pass |
| NS-SCH-022 | SCHEMA §2.3 | `ns_company_series_binding` | `models/scope.py` | Done | table registry | Pass |
| NS-SCH-023 | SCHEMA §2.3 | `ns_branch_series_binding` | `models/scope.py` | Done | table registry | Pass |
| NS-SCH-024 | SCHEMA §2.3 | `ns_tenant_series_override` | `models/scope.py` | Done | table registry | Pass |
| NS-SCH-025 | SCHEMA §2.3 | `ns_assignment_priority` | `models/scope.py` | Done | table registry | Pass |
| NS-SCH-026 | SCHEMA §2.4 | `ns_range_interval` | `models/counter.py` | Done | allocator + rollover | Pass |
| NS-SCH-027 | SCHEMA §2.4 | `ns_range_period` | `models/counter.py` | Done | table registry | Pass |
| NS-SCH-028 | SCHEMA §2.4 | `ns_counter_state` | `models/counter.py` | Done | table registry | Pass |
| NS-SCH-029 | SCHEMA §2.4 | `ns_concurrency_profile` | `models/counter.py` + seed | Done | profiles HTTP | Pass |
| NS-SCH-030 | SCHEMA §2.4 | `ns_buffer_pool` | `models/counter.py` | Done | table registry | Pass |
| NS-SCH-031 | SCHEMA §2.4 | `ns_buffer_lease` | `models/counter.py` + buffer_manager | Done | buffer lease tests | Pass |
| NS-SCH-032 | SCHEMA §2.4 | `ns_buffer_checkpoint` | `models/counter.py` | Done | table registry | Pass |
| NS-SCH-033 | SCHEMA §2.4 | `ns_interval_extension` | `models/counter.py` | Done | extend API | Pass |
| NS-SCH-034 | SCHEMA §2.5 | `ns_allocation` | `models/allocation.py` + allocator | Done | allocate tests | Pass |
| NS-SCH-035 | SCHEMA §2.5 | `ns_allocation_segment_value` | `models/allocation.py` | Done | table registry | Pass |
| NS-SCH-036 | SCHEMA §2.5 | `ns_reservation` | `models/allocation.py` + allocator | Done | reserve tests | Pass |
| NS-SCH-037 | SCHEMA §2.5 | `ns_reservation_item` | `models/allocation.py` | Done | reserve tests | Pass |
| NS-SCH-038 | SCHEMA §2.5 | `ns_void_record` | `models/allocation.py` + allocator | Done | void tests | Pass |
| NS-SCH-039 | SCHEMA §2.5 | `ns_recycle_bin` | `models/allocation.py` + allocator | Done | recycle tests | Pass |
| NS-SCH-040 | SCHEMA §2.5 | `ns_external_intake` | `models/allocation.py` | Done | table registry | Pass |
| NS-SCH-041 | SCHEMA §2.5 | `ns_collision_log` | `models/allocation.py` | Done | table registry | Pass |
| NS-SCH-042 | SCHEMA §2.5 | `ns_manual_override_request` | `models/allocation.py` + API | Done | override API | Pass |
| NS-SCH-043 | SCHEMA §2.5 | `ns_manual_override_grant` | `models/allocation.py` + API | Done | override API | Pass |
| NS-SCH-044 | SCHEMA §2.6 | `ns_legal_policy` | `models/legal.py` + seed | Done | legal HTTP | Pass |
| NS-SCH-045 | SCHEMA §2.6 | `ns_legal_policy_binding` | `models/legal.py` | Done | bindings API | Pass |
| NS-SCH-046 | SCHEMA §2.6 | `ns_threshold_rule` | `models/legal.py` + seed 80%/500 | Done | threshold tests | Pass |
| NS-SCH-047 | SCHEMA §2.6 | `ns_threshold_event` | `models/legal.py` + monitor | Done | threshold breach | Pass |
| NS-SCH-048 | SCHEMA §2.6 | `ns_fiscal_binding` | `models/legal.py` | Done | table registry | Pass |
| NS-SCH-049 | SCHEMA §2.6 | `ns_period_reset_rule` | `models/legal.py` | Done | table registry | Pass |
| NS-SCH-050 | SCHEMA §2.6 | `ns_rollover_job` | `models/legal.py` | Done | table registry | Pass |
| NS-SCH-051 | SCHEMA §2.6 | `ns_rollover_run` | `models/legal.py` + rollover svc | Done | rollover tests | Pass |
| NS-SCH-052 | SCHEMA §2.7 | `ns_changeset` | `models/governance.py` + API | Done | changeset HTTP | Pass |
| NS-SCH-053 | SCHEMA §2.7 | `ns_changeset_item` | `models/governance.py` | Done | changeset HTTP | Pass |
| NS-SCH-054 | SCHEMA §2.7 | `ns_approval` | `models/governance.py` + API | Done | approvals HTTP | Pass |
| NS-SCH-055 | SCHEMA §2.7 | `ns_publish_version` | `models/governance.py` + API | Done | publish HTTP | Pass |
| NS-SCH-056 | SCHEMA §2.7 | `ns_series_package` | `models/governance.py` + installer | Done | package tests | Pass |
| NS-SCH-057 | SCHEMA §2.7 | `ns_series_package_item` | `models/governance.py` | Done | table registry | Pass |
| NS-SCH-058 | SCHEMA §2.7 | `ns_simulate_run` | `models/governance.py` + simulator | Done | simulate tests | Pass |
| NS-SCH-059 | SCHEMA §2.8 | `ns_gap_scan` | `models/ops.py` + gap_scanner | Done | gap scan tests | Pass |
| NS-SCH-060 | SCHEMA §2.8 | `ns_gap_finding` | `models/ops.py` | Done | gap scan tests | Pass |
| NS-SCH-061 | SCHEMA §2.8 | `ns_import_batch` | `models/ops.py` + API | Done | import API | Pass |
| NS-SCH-062 | SCHEMA §2.8 | `ns_import_item` | `models/ops.py` | Done | table registry | Pass |
| NS-SCH-063 | SCHEMA §2.8 | `ns_reconcile_run` | `models/ops.py` + API | Done | reconcile API | Pass |
| NS-SCH-064 | SCHEMA §2.8 | `ns_usage_stats` | `models/ops.py` + API | Done | usage-stats API | Pass |
| NS-SCH-065 | SCHEMA plumbing | `ns_outbox` | `messaging/outbox/models.py` | Done | outbox stream test | Pass |
| NS-SCH-066 | SCHEMA plumbing | `ns_idempotency_key` | `persistence/idempotency.py` | Done | table registry | Pass |
| NS-SCH-067 | SCHEMA plumbing | `ns_series_audit` | `models/plumbing.py` | Done | audit API | Pass |
| NS-SCH-068 | SCHEMA §1 | Schema name `number_series` (never p07) | `schema_constants.py` | Done | `test_schema_constant_is_number_series_not_p07` | Pass |
| NS-SCH-069 | SCHEMA §15 | Seed objects BILTY…BP_CODE | `catalog_store.seed_defaults` | Done | seeds test | Pass |
| NS-SCH-070 | SCHEMA §15 | Legal `IN_GST_TAX_INVOICE` | seed | Done | legal HTTP | Pass |
| NS-SCH-071 | SCHEMA §15 | Profiles GAPLESS_STRICT / BUFFERED_STANDARD | seed | Done | profiles HTTP | Pass |
| NS-SCH-072 | SCHEMA §15 | Threshold defaults 80% / remaining 500 | seed | Done | seeds test | Pass |
| NS-G-001 | GUIDE §2 | Module deps identity/org/configuration | `module.py` | Done | module deps tests | Pass |
| NS-G-002 | GUIDE §11 | Segment engine | `segment_engine.py` | Done | golden format tests | Pass |
| NS-G-003 | GUIDE §11 | Allocator hotspot | `allocator.py` | Done | allocate/idempotent/gapless | Pass |
| NS-G-004 | GUIDE §11 | Buffer manager | `buffer_manager.py` | Done | lease/revoke | Pass |
| NS-G-005 | GUIDE §11 | Gapless lock | `gapless_lock.py` | Done | unit available | Pass |
| NS-G-006 | GUIDE §11 | Check digit Luhn/Mod97 | `check_digit.py` | Done | check digit tests | Pass |
| NS-G-007 | GUIDE §11 | Legal policy guard | `legal_policy_guard.py` | Done | legal tests | Pass |
| NS-G-008 | GUIDE §11 | Rollover | `rollover.py` | Done | rollover tests | Pass |
| NS-G-009 | GUIDE §11 | Gap scanner | `gap_scanner.py` | Done | gap anomaly test | Pass |
| NS-G-010 | GUIDE §11 | Simulator | `simulator.py` | Done | simulate no mutation | Pass |
| NS-G-011 | GUIDE §11 | Package installer | `package_installer.py` | Done | checksum fail/success | Pass |
| NS-G-012 | GUIDE §10 | Permissions `number_series.*` | `permissions/catalog.py` | Done | permissions gate | Pass |
| NS-G-013 | GUIDE §13 | Outbox stream `jesloterp:number_series:outbox` | outbox models | Done | stream constant test | Pass |
| NS-G-014 | GUIDE §4.4 | CONTINUOUS / BUFFERED / EXTERNAL / HYBRID | enums + allocator | Done | mode tests | Pass |
| NS-G-015 | GUIDE §5.3 | Idempotent allocate | allocator + store | Done | idempotent tests | Pass |
| NS-G-016 | GUIDE §6 | Tax invoice gapless + forbid_reuse | legal guard + void/recycle | Done | recycle forbidden | Pass |
| NS-G-017 | GUIDE §15 DoD | Peek never mutates | allocator.peek | Done | peek test | Pass |
| NS-API-001 | API §6.1 | POST `/peek` | `routers/runtime.py` | Done | peek permission + OpenAPI | Pass |
| NS-API-002 | API §6.2 | POST `/allocate` + Idempotency-Key | runtime | Done | HTTP allocate replay | Pass |
| NS-API-003 | API §6.3 | Reservations commit/release | runtime | Done | reserve HTTP | Pass |
| NS-API-004 | API §6.4 | allocate-batch | runtime | Done | batch HTTP | Pass |
| NS-API-005 | API §6.5 | allocate-manual | runtime | Done | manual HTTP | Pass |
| NS-API-006 | API §6.6 | validate | runtime | Done | validate HTTP | Pass |
| NS-API-007 | API §6.7 | void / recycle | runtime | Done | void+forbid recycle | Pass |
| NS-API-008 | API §6.8 | get/list allocations | runtime | Done | get allocation | Pass |
| NS-API-009 | API §7 | simulate | runtime | Done | simulate HTTP | Pass |
| NS-API-010 | API §8 | objects/definitions/templates | catalog router | Done | catalog HTTP | Pass |
| NS-API-011 | API §9 | assignments/intervals/profiles | assignments router | Done | OpenAPI needles | Pass |
| NS-API-012 | API §10 | legal + thresholds | admin_ops | Done | legal/threshold HTTP | Pass |
| NS-API-013 | API §11 | changeset/publish | admin_ops | Done | changeset HTTP | Pass |
| NS-API-014 | API §12 | packages install | admin_ops + installer | Done | package HTTP | Pass |
| NS-API-015 | API §13 | rollover/gap/reconcile/usage | admin_ops | Done | ops HTTP | Pass |
| NS-API-016 | API §14 | imports | admin_ops | Done | OpenAPI path | Pass |
| NS-API-017 | API §15 | manual overrides | admin_ops | Done | OpenAPI path | Pass |
| NS-API-018 | API §16 | buffers + internal lease | admin_ops | Done | buffer tests | Pass |
| NS-API-019 | API §17 | audit | admin_ops | Done | audit HTTP | Pass |
| NS-API-020 | API §18 | internal allocate/health | admin_ops | Done | internal allocate + health | Pass |
| NS-API-021 | API §4 | NS_* error codes | `domain/exceptions.py` + handlers | Done | 404/422/403 tests | Pass |
| NS-API-022 | API §1 | Public `/api/v1/number-series` + internal base | `api_v1.py` | Done | OpenAPI needles | Pass |
| NS-WIR-001 | sample / mirror p06 | ModulePlugin wiring | `module.py` | Done | module tests | Pass |
| NS-WIR-002 | LOCALIZATION_GUIDE wiring | apps/api/main.py register NumberSeriesModule + exception handlers | apps/api/main.py | Implemented | test_load_modules_includes_number_series | Passed |
| NS-WIR-003 | NUMBER_SERIES_SCHEMA DDL/RLS | Alembic create + FORCE RLS | `c1d2e3f4a5b6` / `d2e3f4a5b6c7` | Implemented | test_all_67_number_series_tables_registered | Passed |
| NS-WIR-004 | alembic/env.py | Import p07 persistence models | alembic/env.py | Implemented | alembic upgrade head | Passed |

**Coverage summary:** 67/67 tables registered · Guide §11 services implemented · API public+internal routes present (69 OpenAPI paths) · pytest green · Alembic head `d2e3f4a5b6c7` · ModuleRegistry Live.
