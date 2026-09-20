# Licensing Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p26_licensing/tests -q --tb=short` → **20 passed**; **94** `license_*` tables; load order after p25; outbox `jesloterp:licensing:outbox`.

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| LIC-G-01 | V2 GUIDE §1 | Commercial SoR: catalog, plans, subs, entitlements, meters, checkout, billing refs, marketplace/on-prem | `LicensingCatalogStore` + schema `licensing` | Implemented | module tables + health | PASS |
| LIC-G-02 | V2 GUIDE §1 | Does not own p12 flags, p01 RBAC, payment/tax/GL ledgers | UUID refs / payment_reference only | Implemented | checkout payment_reference | PASS |
| LIC-G-03 | V2 GUIDE §1 | `effective = entitled AND flag AND permitted` | `POST /effective/check` | Implemented | entitlement effective | PASS |
| LIC-G-04 | V2 GUIDE §2 | No cross-schema FKs | UUID columns on ORM | Implemented | table inventory | PASS |
| LIC-G-05 | V2 GUIDE §2 | Published plan/price/addon versions immutable | `put_plan_version_features` / publish | Implemented | publish then 409 | PASS |
| LIC-G-06 | V2 GUIDE §2 | Entitlements compiled; override > contract > addon > plan | `compile_entitlement` + provenance cols | Implemented | compile + override deny | PASS |
| LIC-G-07 | V2 GUIDE §2 | Tenant from JWT/test user, not X-Tenant-Id alone | `_tenant(actor)` | Implemented | subscribe tenant isolation | PASS |
| LIC-G-08 | V2 GUIDE §2 | Money `Numeric(18,6)` | ORM money columns | Implemented | table models | PASS |
| LIC-G-09 | V2 GUIDE §2 | Payment/tax are coordination refs | `license_payment_reference` | Implemented | checkout confirm | PASS |
| LIC-G-10 | V2 GUIDE §2 | Usage ingest idempotent | `ingest_usage` DUPLICATE/200 | Implemented | usage replay | PASS |
| LIC-G-11 | V2 GUIDE §2 | Mount `/api/v1/licensing` not bare `/products` | `api_v1.py` prefix | Implemented | all API tests | PASS |
| LIC-G-12 | V2 GUIDE §6 | Permissions `licensing.*` | `require_licensing_permission` | Implemented | catalog 403 | PASS |
| LIC-G-13 | V2 GUIDE §6 | FORCE RLS tenant commercial/usage/entitlement | Alembic `f26b1c2d3e4f` | Implemented | migration present | PASS |
| LIC-G-14 | V2 GUIDE §8 | Outbox domain events + stream | `_emit` + `jesloterp:licensing:outbox` | Implemented | outbox stream test | PASS |
| LIC-S-01 | V2 SCHEMA §1 | Schema `licensing` (never p26/p04); `license_*` | `LICENSING_SCHEMA` | Implemented | table names | PASS |
| LIC-S-02 | V2 SCHEMA §2 + v1 `__tablename__` | All v1 tablenames (94); no `license_entitlement_source` | ORM models | Implemented | `test_licensing_module_tables_match_v1_list` | PASS |
| LIC-S-03 | V2 SCHEMA §3 | Enums subscription/plan/feature/source/usage/dunning/invoice | `domain/enums.py` | Implemented | API payloads | PASS |
| LIC-S-04 | V2 SCHEMA §4 | Plan version immutable; subscription version; entitlement provenance; meter idempotency; payment refs | store + models | Implemented | publish/If-Match/usage | PASS |
| LIC-S-05 | V2 SCHEMA §5 | Compile model + provenance on feature rows | `compile_entitlement` | Implemented | compile source_type | PASS |
| LIC-S-06 | V2 SCHEMA §6 | FORCE RLS tenant tables | Alembic FORCE list | Implemented | migration | PASS |
| LIC-S-07 | V2 SCHEMA §7 | Seed INR/USD, jesloterp.saas, starter/growth/enterprise, meters, trial, pack, perms | `seed_defaults` + Alembic perms | Implemented | catalog + subscribe | PASS |
| LIC-A-00 | V2 API §0 | Envelope + AUTH/FORBIDDEN/NOT_ENTITLED/NOT_FOUND/CONFLICT/VERSION_CONFLICT/PAYMENT_REQUIRED/VALIDATION/QUOTA/RATE/SUBSCRIPTION_BLOCKED | `domain/exceptions.py` | Implemented | all API families | PASS |
| LIC-A-01 | V2 API §1 | Products/features/modules/currencies/units | `/products` `/features` `/modules` `/currencies` `/units` | Implemented | catalog family | PASS |
| LIC-A-02 | V2 API §2 | Plans/prices/addons + publish freeze 409 | `/plans*` `/prices` `/addons*` | Implemented | plans family | PASS |
| LIC-A-03 | V2 API §3 | Subscriptions change-plan/pause/resume/cancel/schedules + If-Match | `/subscriptions*` | Implemented | subscription family | PASS |
| LIC-A-04 | V2 API §4 | Compile/overrides/token/check-feature/check-limit/effective/introspect | `/entitlement*` `/effective/check` | Implemented | entitlement family | PASS |
| LIC-A-05 | V2 API §5 | Usage events/batch/ledger/reset/overage/rating/meters | `/usage*` `/meters` | Implemented | usage family | PASS |
| LIC-A-06 | V2 API §6 | Checkout/quotes/coupons | `/checkouts*` `/quotes*` `/coupons` | Implemented | checkout family | PASS |
| LIC-A-07 | V2 API §7 | Invoices/dunning/payment-refs | `/invoices*` `/dunning*` `/payment-references` | Implemented | billing family | PASS |
| LIC-A-08 | V2 API §8 | Contracts/marketplace/onprem activate+heartbeat | `/contracts` `/marketplace*` `/onprem*` | Implemented | onprem family | PASS |
| LIC-A-09 | V2 API §9 | Webhooks/packages/signing-keys/health live+ready | `/webhooks` `/packages` `/signing-keys` `/health/*` | Implemented | admin + health | PASS |
| LIC-A-10 | V2 API §9 + brief | Internal `/internal/v1/licensing/*` aliases + health | `api_v1.py` | Implemented | internal alias + health | PASS |
| LIC-A-11 | V2 API §0 / hard rules | Quota 429; dunning 423; past_due 402; deny 403; If-Match 409 | store guards | Implemented | dedicated hard-rule tests | PASS |
| LIC-A-12 | V2 API health like p21 | LIVE ≠ READY; ready 503 when deps down | `health_live` / `health_ready` | Implemented | health live not ready | PASS |
| LIC-M-01 | brief | Package `platforms.p26_licensing`; ModulePlugin after DashboardModule | `main.py` | Implemented | load after p25 | PASS |
| LIC-M-02 | brief | Alembic f26a revises f25b; f26b FORCE RLS | `alembic/versions/f26*` | Implemented | revision ids | PASS |
| LIC-M-03 | brief | Depends p01/p02/p03/p04/p12 | `LicensingModule.dependencies` | Implemented | deps test | PASS |
| LIC-SOR-01 | TASK-SOR-022 | Product/subscription/entitlement HTTP → Postgres; empty `[]` | `catalog_repository` + `require_licensing_access` | Implemented | persist-then-fetch + db-first empty | PASS |
| LIC-SOR-02 | TASK-SOR-022 | PSP ports; no fake live payment ids | Stripe/Razorpay `PROVIDER_PENDING` | Implemented | live class + factory stub under pytest | PASS |
