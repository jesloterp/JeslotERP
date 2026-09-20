# JeslotERP Localization Platform (p06) — Implementation Record

**Date:** 2026-09-12  
**Package:** `platforms.p06_localization`  
**PostgreSQL schema:** `i18n`  
**Source of truth reviewed:** `LOCALIZATION_GUIDE.md`, `LOCALIZATION_SCHEMA.md`, `LOCALIZATION_API.md`, `docs/sample.md`

---

## 1. Overview & Objective

Deliver the backend localization control plane for JeslotERP: effective ICU resolve with ETag, CLDR-class locale catalog, authoring APIs, TMS workflow, glossary/TM, language packs, publish governance, coverage/lint/pseudo, MT assist hooks, XLIFF/JSON import-export, audit, internal mesh APIs, permissions (`i18n.*`), full ORM coverage for 62 schema tables, Alembic DDL + FORCE RLS, and ModuleRegistry wiring. HTTP mirrors p05_metadata patterns (`/api/v1/i18n`, `/internal/v1/i18n`, auth/access/permissions, `register_localization_exception_handlers`).

---

## 2. All 3 Source Documents Reviewed

| Document | Version | Review outcome |
| --- | --- | --- |
| `LOCALIZATION_GUIDE.md` | 1.0 | Architecture, effective model, permissions, DoD — mapped to code/tests |
| `LOCALIZATION_SCHEMA.md` | 1.0 | 62-table contract — SQLAlchemy models + Alembic `a9b0c1d2e3f4` / RLS `b0c1d2e3f4a5` |
| `LOCALIZATION_API.md` | 1.0 | Public + internal routes — routers registered and contract-tested |

---

## 3. Existing Backend Architecture Reviewed

- **Module plugin:** `LocalizationModule` in `platforms/p06_localization/module.py`; registered in `apps/api/main.py` after p01–p05.
- **HTTP layering:** Routers under `infrastructure/http/routers/*`, composed in `infrastructure/http/api_v1.py`.
- **Auth:** JWT via `dependencies/auth.py`; permission gate via `dependencies/access.py` + `permissions.py`.
- **Standard responses:** `shared.api.response.StandardResponse`.
- **Persistence:** ORM on schema `i18n`; runtime authoring/TMS may use in-memory `LocalizationCatalogStore` with ORM/migrations for production DDL.
- **Alembic:** Models imported in `alembic/env.py`; create + RLS revisions after metadata head `f8a9b0c1d2e3`.

---

## 4. Requirements Identified

Requirements extracted into **103 RTM rows** (see `LOCALIZATION_RTM.md`): 62 schema tables, schema cross-cutting (RLS/seed/mixin/migration), guide items including module wiring, and API/checklist items.

---

## 5. Requirement-by-Requirement Implementation

| Area | Implementation | Verification |
| --- | --- | --- |
| Effective resolve | `effective_resolver.py`, `fallback_chain.py`, `effective.py` | Unit + HTTP (pack ETag/304, format, locale shell) |
| ICU engine | `icu_engine.py` en/hi/ar plural + select | `tests/unit/icu/test_icu_engine.py` |
| Catalog authoring | `catalog.py` + `catalog_store` | API contract tests |
| Locales / CLDR | `locales.py` | Contract + fallback PUT |
| Overrides / prefs | `overrides.py` | Contract tests |
| TMS | `tms.py` + glossary lint on submit | Contract flow test |
| Glossary / TM | `glossary_tm.py`, `glossary_lint.py`, `glossary_linter.py` | Unit + API lint 422 |
| Packs / publish | `packages_publish.py`, `package_installer.py`, `publisher.py` | Unit + API install/publish |
| Coverage / pseudo | `coverage.py`, `quality.py` | Unit + API scan |
| MT | `mt.py`, `mt_orchestrator.py` | API contract |
| Import/export | `import_export.py`, `adapters/xliff.py` | API contract |
| Internal mesh | `internal.py` + `X-Internal-Token` | Contract tests |
| Permissions | `application/permissions/catalog.py` + Alembic seed | `test_permissions_gate.py` |
| ORM 62 tables | `infrastructure/persistence/models/*` | `test_all_62_i18n_tables_registered` |
| Module wiring | `apps/api/main.py` | `test_load_modules_includes_localization` |
| DDL + RLS | Alembic `a9b0c1d2e3f4`, `b0c1d2e3f4a5` | Migration chain head `b0c1d2e3f4a5` |

---

## 6. Files/Modules/Services Created or Modified

**Module:** `platforms/p06_localization/module.py`, `infrastructure/module.py`

**Domain:** `domain/enums.py`, `domain/exceptions.py`, `domain/policies/publish_gates.py`

**Application:** `application/services/*` (catalog_store, effective_resolver, icu_engine, fallback_chain, glossary_lint, glossary_linter, coverage, publisher, package_installer, pseudo, tm_matcher, mt_orchestrator), `application/permissions/catalog.py`, `application/errors.py`

**Infrastructure HTTP:** `infrastructure/http/api_v1.py`, `routers/*`, `dependencies/auth.py`, `dependencies/permissions.py`, `dependencies/access.py`, `exception_handlers.py`, `route_common.py`

**Persistence:** `infrastructure/persistence/models/*`, `idempotency.py`, `resolve_cache.py`, `schema_constants.py`

**Messaging:** `infrastructure/messaging/outbox/*`

**API wiring:** `apps/api/main.py` — `LocalizationModule` + `register_localization_exception_handlers`

**Alembic:** `alembic/env.py` model import; `a9b0c1d2e3f4_create_i18n_schema.py`; `b0c1d2e3f4a5_enable_i18n_rls.py`

**Tests:** `platforms/p06_localization/tests/unit/{icu,resolver,glossary,coverage,publish,packages,api,module,permissions,fallback}/`

**Docs:** `LOCALIZATION_RTM.md`, `LOCALIZATION_IMPLEMENTATION_RECORD.md`; GUIDE/SCHEMA/API status; `PLATFORM_REGISTRY.md` Live + phase checkbox

---

## 7. Database Changes & Migrations

| Revision | Purpose |
| --- | --- |
| `a9b0c1d2e3f4` | `CREATE SCHEMA i18n`; `create_all` for 62 tables; seed languages/scripts/territories/locales/fallback/channels/namespaces/critical EN messages/format profiles (en-IN, hi-IN)/lint rules; seed `i18n.*` permissions + admin role grants |
| `b0c1d2e3f4a5` | ENABLE + FORCE RLS on 47 tenant-scoped tables; `i18n.current_tenant_id()` / `i18n.rls_bypass()`; policy allows bypass OR `tenant_id IS NULL` OR matching tenant |

**Down revisions:** `a9b0…` revises `f8a9b0c1d2e3` (metadata RLS); `b0c1…` revises `a9b0…`.

---

## 8. APIs/Endpoints Implemented or Updated

- **Public prefix:** `/api/v1/i18n` — effective, locales, catalog, overrides, TMS, glossary/TM, packages/publish, coverage/lint/pseudo, MT, import/export, audit, lookups, health, version.
- **Internal prefix:** `/internal/v1/i18n` — hydrate-labels, hydrate-template, effective/formats, cache/invalidate, health.
- **OpenAPI path count (localization router only):** **88** routes (verified via `FastAPI().include_router(LocalizationModule().router)`).

---

## 9. Business Rules & Workflows Implemented

- Layered message merge with debug provenance (`resolve_messages` origins).
- ICU plural rules for `en`, `hi-IN`, `ar-AE`; select support in `format_message`.
- Glossary lint blocks forbidden terms on TMS submit (`I18N_GLOSSARY_VIOLATION` → 422).
- Coverage BLOCKER gate on publish when enabled.
- Package install checksum verification (`I18N_CHECKSUM_MISMATCH` → 409).
- Publish increments immutable bundle version + audit trail.
- MT assist-only (suggestions/proposals; no auto-publish).

---

## 10. Validation, Permissions & Error Handling

- Permission codes in `I18N_PERMISSIONS` including `i18n.*`; seeded in Alembic.
- `require_i18n_access` fail-closed; platform admin role bypass in `require_i18n_permission`.
- `register_localization_exception_handlers` maps `I18nError` to 404/403/409/422 as documented; registered on FastAPI app.

---

## 11. Integrations Implemented

- Internal hydrate-labels for p05 metadata label keys (contract tested).
- Internal hydrate-template stub for p15 notification path.
- MT provider registry uses `secret_ref_key` only (no secrets in i18n schema).
- Depends on `p01_identity` + `p03_configuration` via ModuleRegistry topo order.

---

## 12. Test Cases Created for Each Functionality

≥2 variations per area where applicable: `icu/`, `resolver/`, `glossary/`, `coverage/`, `publish/`, `packages/`, `api/` (route surface + area contracts), `module/` (incl. load_modules wiring), `permissions/`, `fallback/`, plus `test_full_platform_contract.py`.

---

## 13. Test Execution Results

```text
pytest platforms/p06_localization/tests -q
71 passed

pytest platforms/p05_metadata/tests platforms/p04_business_partner/tests/unit core shared -q
111 passed
```

Alembic head (script): `b0c1d2e3f4a5`. Local DB was at older revision `f2a3b4c5d6e7`; full `upgrade head` blocked by a pre-existing BP seed `row_version` issue in `a3b4c5d6e7f8` (out of p06 scope). New i18n revisions are chained after metadata `f8a9b0c1d2e3`.

---

## 14. Requirements Traceability Matrix (RTM)

See `docs/platforms/06_localization/LOCALIZATION_RTM.md` — **103 rows**, SCHEMA-T01..T62 + guide + API + migration/wiring mappings.

---

## 15. Issues Found & How They Were Resolved

- Merged parallel p06 skeleton with HTTP deliverable: aligned router prefixes under `/api/v1/i18n`, unified exception handlers, fixed TMS proposal UUID lookup, extended publisher/package_installer helpers for tests.
- Parent wiring gap closed: ModuleRegistry registration, exception handlers, Alembic model import, create+RLS migrations, registry phase checkbox.
- Fallback seed SQL corrected to `I18nPlatformBase` columns (no `message_lifecycle`).

---

## 16. Regression/Existing Functionality Verification

- Localization suite green after wiring.
- `load_modules()` returns topo order including `p06_localization` after identity/configuration/metadata.
- Existing p04/p05/core/shared tests re-run for regression.

---

## 17. Final Coverage & Completion Status

| Metric | Result |
| --- | --- |
| Tests (p06) | **71 passed** |
| Regression (p04/p05/core/shared) | **111 passed** |
| OpenAPI paths | **88** |
| RTM rows | **103** |
| Schema tables (ORM) | **62** |
| Alembic head (script) | `b0c1d2e3f4a5` |
| Module load order | p01→p02→p03→p04→p05→**p06** |
| Registry | **Live** + phase checkbox complete |

---

## 18. Remaining Issues or Limitations

- HTTP messages + ICU overlays persist on `AsyncSession`; empty list is `[]`. MEMORY/`LocalizationCatalogStore` remains the TestClient/AsyncMock double. TMS/glossary/publish still memory. Not Production.
- Full XLIFF round-trip binary validation is stubbed at HTTP layer; adapter module exists for expansion.
- Apply Alembic migrations in each environment (`alembic upgrade head`) before production use of Postgres-backed i18n tables.
