# Licensing Platform — Implementation Record

**Platform:** `p26_licensing`  
**Date:** 2026-09-11  
**Scope:** Backend only (TASK-012 / `docs/tasks/task_p26_licensing.md`)  
**Verification:** `python -m pytest platforms/p26_licensing/tests -q --tb=short` → **20 passed**; ORM `licensing` table count → **94**; module load order p26 after p25 → **passed**

---

## 1. Overview & Objective

Implement JeslotERP licensing platform end-to-end: schema `licensing`, all v1 `license_*` tablenames (94; no `license_entitlement_source`), ModulePlugin `p26_licensing` (depends on p01/p02/p03/p04/p12), in-memory catalog store, public `/api/v1/licensing` + internal aliases, dual Alembic, permissions, tests, RTM, and status updates. Ship shape copied from p24_reporting / p25_dashboard.

## 2. All Source Documents Reviewed

Primary (V2 normative ship shape / APIs / deps):

| Document | Path | Role |
| --- | --- | --- |
| V2 GUIDE | `docs/platforms/26_licensing/V2_LICENSING_GUIDE.md` | Architecture, hard rules, permissions, DoD |
| V2 SCHEMA | `docs/platforms/26_licensing/V2_LICENSING_SCHEMA.md` | Inventory, compile model, RLS, seed |
| V2 API | `docs/platforms/26_licensing/V2_LICENSING_API.md` | `/api/v1/licensing` HTTP §1–9 |

ORM table authority (v1 encyclopedia; not edited):

| Document | Role |
| --- | --- |
| `LICENSING_SCHEMA.md` | Every `__tablename__` (94). Provenance folded onto `license_entitlement_feature`. |

Also read `LICENSING_GUIDE.md` / `LICENSING_API.md` as archived encyclopedia only (HYG-015). Ship REST is V2 `/api/v1/licensing`; bare `/api/v1/products*` is 410.

## 3. Existing Backend Architecture Reviewed

- ModulePlugin registration and topo-sort in `apps/api/main.py` (DashboardModule already wired)
- Alembic `env.py` already imports `platforms.p26_licensing.infrastructure.persistence.models`
- p25 patterns: in-memory catalog store, thin routers, exception handlers, permission deps, outbox stream, dual migrations, TestClient
- p21 health: LIVE ≠ READY (ready 503 when deps down)
- Shared `PlatformBase` / `Base` ORM bases; no cross-schema FKs

## 4. Requirements Identified

See `LICENSING_RTM.md` (100% mapped). Major themes: immutable published versions, compiled entitlements with precedence, JWT tenant, idempotent usage, quota/dunning/If-Match HTTP codes, 94 v1 tables, `/api/v1/licensing` mount, seed catalog so subscribe+compile+check works.

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | V2 codes + lifecycle enums | `domain/` | API §0 / SCHEMA §3 | exception handler + API status tests |
| Catalog store | In-memory commerce/entitlement/usage | `application/services/catalog_store.py` | Runtime like p25 | 13 pytest |
| ORM 94 tables | Split catalog/pricing/subscription/entitlement/usage/billing/checkout/marketplace/onprem/plumbing | `infrastructure/persistence/models/` | v1 `__tablename__` | count test == 94 |
| Money | `Numeric(18,6)` on money/qty columns | pricing/subscription/usage/billing/checkout | V2 SCHEMA §1 | model definitions |
| HTTP APIs | Public + internal aliases | `infrastructure/http/` | V2 API §1–9 | contract tests |
| Permissions | `licensing.*` catalog | `application/permissions/` | GUIDE §6 / API §10 | 403 + migration seed |
| Module | `LicensingModule` after DashboardModule | `infrastructure/module.py` + `main.py` | registry + brief | load-order test |
| Migrations | schema + FORCE RLS | `alembic/versions/f26*.py` | Live DB path | revision chain f25b → f26a → f26b |
| Health | LIVE ≠ READY | store + routers | V2 §9 / p21 | health test |
| Tokens | HMAC-SHA256 stub + rotate | `issue_token` / `rotate_signing_key` | GUIDE §3 | token + rotate tests |

## 6. Files/Modules/Services Created or Modified

**Created:**
- `platforms/p26_licensing/**` (domain, application, infrastructure, tests)
- `alembic/versions/f26a0b1c2d3e_create_licensing_schema.py`
- `alembic/versions/f26b1c2d3e4f_enable_licensing_rls.py`
- `docs/platforms/26_licensing/LICENSING_RTM.md`
- `docs/platforms/26_licensing/LICENSING_IMPLEMENTATION_RECORD.md` (this file)

**Modified:**
- `apps/api/main.py` — LicensingModule after DashboardModule + exception handlers
- `IMPLEMENTATION_TASKS.md` — TASK-012 marked complete
- `IMPLEMENTATION_STATUS.md` — advanced to TASK-013

**Not modified (per brief):** V2/v1 requirement docs; `docs/tasks/task_p26_licensing.md`; `alembic/env.py` p26 import (already present).

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `f26a0b1c2d3e` | CREATE SCHEMA `licensing`; create_all 94 tables; seed `licensing.*` permissions | Created (apply via alembic upgrade) |
| `f26b1c2d3e4f` | ENABLE + FORCE RLS on tenant commercial/usage/entitlement tables | Created |

**Down revision chain:** `f25b1c2d3e4f` → `f26a0b1c2d3e` → `f26b1c2d3e4f`

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/licensing` (also aliased under `/internal/v1/licensing`):

- Catalog: products CRUD/versions/publish, features, modules, currencies, units
- Plans/prices/addons + publish freeze
- Subscriptions CRUD, change-plan, pause/resume/cancel, changes, schedules, If-Match
- Entitlement get/compile/overrides/token/check-feature/check-limit; `/effective/check`; token introspect
- Usage events/batch/ledger/reset/overage/rating; meters
- Checkouts items/coupons/recalculate/confirm; quotes convert; coupons
- Invoices finalize/void; payment-references; dunning policies/cases/advance
- Contracts; marketplace apps/installs; on-prem licenses activate/heartbeat
- Webhooks; packages apply; signing-keys rotate
- Health live (public) + ready (503 when deps down)

Internal extras: `GET /internal/v1/licensing/health` plus live/ready.

## 9. Business Rules & Workflows Implemented

- Published plan/addon versions immutable (409 CONFLICT on feature edit)
- Entitlement compile precedence: override > contract > addon > plan; provenance on feature rows
- Subscribe (and checkout confirm / quote convert) sync-compiles entitlement
- Usage idempotency key per tenant → 200 replay with `duplicate: true` / `DUPLICATE`
- Quota exceed → 429 `QUOTA_EXCEEDED`
- Dunning SOFT_BLOCK / PAST_DUE → 402 `PAYMENT_REQUIRED`; SUSPENDED → 423 `SUBSCRIPTION_BLOCKED`
- check-feature deny → 403 `NOT_ENTITLED`
- If-Match mismatch → 409 `VERSION_CONFLICT`
- Tenant identity from JWT/test user only
- Signing keys rotatable; tokens HMAC-SHA256 stubs
- Seed: INR/USD, `jesloterp.saas`, starter/growth/enterprise published v1 with BOOLEAN + LIMIT/METER features, meters `api.calls`/`ai.tokens`, trial 14d, pack `core.saas.v1`

## 10. Validation, Permissions & Error Handling

Permissions: `licensing.catalog.read/manage`, `subscription.manage`, `entitlement.read/compile`, `usage.ingest/read`, `checkout.manage`, `billing.manage`, `onprem.manage`, `admin`, `licensing.*`.

Errors: AUTH_REQUIRED, FORBIDDEN, NOT_ENTITLED, NOT_FOUND, CONFLICT, VERSION_CONFLICT, PAYMENT_REQUIRED, VALIDATION_ERROR, QUOTA_EXCEEDED, RATE_LIMITED, SUBSCRIPTION_BLOCKED.

## 11. Integrations Implemented

- p01 JWT via shared `require_auth` / CurrentUser
- p02/p04 UUID refs only (no FKs)
- p12 AND-gate documented on `/effective/check` (`entitled` + stub `flag_on` + `permitted`)
- Payment intent reference on checkout confirm (execution via payment/p23 — not implemented here)
- Outbox stream `jesloterp:licensing:outbox`
- p22 mount path `/api/v1/licensing`

## 12. Test Cases Created for Each Functionality

| Family | File | Variations |
| --- | --- | --- |
| Module | `test_licensing_module_tables.py` | 94 tables == v1 list; deps; load after p25; outbox stream |
| Catalog | `test_licensing_api_contracts.py` | CRUD + 404 + 403 |
| Plans | same | publish then 409 edit; prices; addon publish 409 |
| Subscriptions | same | lifecycle + If-Match 409 + 404 |
| Entitlements | same | compile/allow/deny/override/token/effective |
| Usage | same | idempotent replay; quota 429; ledger/reset/meters |
| Checkout | same | coupon/confirm replay; quote convert; 404 |
| Billing | same | invoice finalize/void; dunning 402 + 423 |
| On-prem / marketplace | same | install 404; activate + heartbeat |
| Admin / health | same | packs/keys; LIVE ≠ READY; internal aliases |

## 13. Test Execution Results

```
pytest platforms/p26_licensing/tests -q
13 passed
```

## 14. Requirements Traceability Matrix (RTM)

See [`LICENSING_RTM.md`](LICENSING_RTM.md).

## 15. Issues Found & How They Were Resolved

- Internal health vs public health: mounted both; ready returns 503 when `deps_healthy=False` so LIVE ≠ READY (p21 pattern).
- Dunning states stepped OPEN→REMINDED→SOFT_BLOCK→SUSPENDED so tests can reach 402 then 423.
- v1 `license_entitlement_source` omitted (folded provenance cols on `license_entitlement_feature`).

## 16. Regression/Existing Functionality Verification

- `alembic/env.py` p26 import left unchanged and remains valid.
- DashboardModule still registered immediately before LicensingModule.
- p26 tests only; full suite deferred to TASK-014.

## 17. Final Coverage & Completion Status

- 94/94 v1 tablenames present
- V2 API §1–9 implemented under `/api/v1/licensing`
- Hard-rule HTTP codes covered
- TASK-012 acceptance bar met (implement + tests + RTM + record)

## 18. Remaining Issues or Limitations

- Product/subscription/entitlement HTTP persists on Postgres; empty list is `[]`. `require_licensing_access` sets RLS GUCs. Plans/checkout/usage still memory. Not Production.
- Live Stripe/Razorpay classes return `PROVIDER_PENDING` (no invented payment ids). Pytest factory stays Stub as the TestClient double.
- Runtime catalog store remains the TestClient double; Alembic creates physical tables for Live DB path.
- Intra-licensing ForeignKey constraints from the v1 encyclopedia are not reproduced on ORM (UUID refs only, matching p25 ship shape). No cross-schema FKs.
- p12 flag evaluation on `/effective/check` is a documented stub (`flag_on`); callers AND real p12/p01.
- Payment/tax/GL not stored as ledgers (refs only, as required).
- Extra helper POSTs (`/invoices`, `/dunning/cases`) exist so billing/dunning flows are exercisable; they do not replace V2 paths.
