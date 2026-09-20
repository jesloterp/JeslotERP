# Integration Platform — Implementation Record

**Platform:** `p23_integration`  
**Date:** 2026-09-12  
**Verification:** `python -m pytest platforms/p23_integration/tests -q --tb=short` → **22 passed**; 65 `integration` tables; load order after p22.

## 1. Overview & Objective

External system connectivity control plane: schema `integration`, connector catalog, tenant connections (secret refs only), mappings, pipelines, outbound delivery with retry/circuit/DLQ, inbound webhooks (no JWT), partner profiles, reconcile/sync-state, packs/analytics. Alembic f23a/f23b. Outbound messages + delivery attempts persist on Postgres (TASK-SOR-021). Connector adapters stay ports (`stub` / `PROVIDER_PENDING`). Catalog/mappings/pipelines/DLQ/inbound still memory.

## 2. Source documents reviewed

`INTEGRATION_GUIDE.md`, `INTEGRATION_SCHEMA.md`, `INTEGRATION_API.md` under `docs/platforms/23_integration/` plus `docs/tasks/task_p23_integration.md`. GUIDE / SCHEMA / API updated to **SoR-Live** for TASK-SOR-021.

## 3. Existing backend architecture reviewed

p22_api ModulePlugin, in-memory catalog store, exception handlers from p05 auth, `require_internal_token`, dual Alembic schema+RLS, TestClient factory. Same layout copied for p23. Depends on p13_event_bus, p14_messaging, p22_api.

## 4. Requirements identified

See `INTEGRATION_RTM.md` (100% of GUIDE / SCHEMA / API items mapped).

## 5. Requirement-by-requirement implementation

HTTP deliver / internal deliver dual-write `integration_outbound_message` + `integration_delivery_attempt` when `session` is `AsyncSession`. Empty attempt list is `[]`. `internal_deliver` calls `get_connector_adapter` (REST stub records the attempt; unknown kinds return `PROVIDER_PENDING`). Connections/webhooks already dual-write. Catalog, mappings, pipelines, DLQ, inbound receptions stay in `IntegrationCatalogStore` for TestClient.

## 6. Files created or modified

**Created**

- `platforms/p23_integration/` domain, application, infrastructure HTTP/module/adapters, tests
- `platforms/p23_integration/infrastructure/persistence/schema_constants.py` (`INTEGRATION_SCHEMA = "integration"`)
- `integration_outbox` + `integration_idempotency_key`; models `__init__` imports all 65 classes
- `alembic/versions/f23a0b1c2d3e_create_integration_schema.py`
- `alembic/versions/f23b1c2d3e4f_enable_integration_rls.py`
- `docs/platforms/23_integration/INTEGRATION_RTM.md`
- this record

**Modified**

- `apps/api/main.py` — `IntegrationModule` after `ApiModule`; exception handlers
- `alembic/env.py` — import p23 models
- `IMPLEMENTATION_TASKS.md`, `IMPLEMENTATION_STATUS.md`

**TASK-SOR-021**

- `platforms/p23_integration/infrastructure/persistence/repositories/outbound_repository.py`
- `platforms/p23_integration/application/ports/connector_adapter.py`
- `platforms/p23_integration/application/services/adapter_factory.py`
- `platforms/p23_integration/infrastructure/http/dependencies/access.py`
- GUIDE / SCHEMA / API / RTM / this record → **SoR-Live**

## 7. Database changes & migrations

Schema `integration`, 63 domain `integration_*` tables + `integration_outbox` + `integration_idempotency_key` = **65**. FORCE RLS on tenant connections, deliveries, receptions, DLQ, partner profiles (and related tenant-scoped children). Permissions `integration.*` seeded in f23a. No cross-schema FKs.

## 8. APIs

Public `/api/v1/integration` (API §1–10). Public webhooks `POST/GET /api/v1/hooks/{tenant_slug}/{endpoint_key}` (no JWT). Internal deliver/complete/inbound complete at `/api/v1/integration/internal/*` and `/internal/v1/integration/*`. Health at `/api/v1/integration/health` and `/internal/v1/integration/health`.

## 9. Business rules

- Secrets never returned on GET — `secret_ref` / `vault_path` only; webhook plaintext secret once on create/rotate then hash/ref only
- Circuit open → HTTP 423 `CIRCUIT_OPEN` on deliver
- Mapping test mismatch/missing required → HTTP 422 `MAPPING_FAILED`
- Webhook bad/missing signature → HTTP 401 `WEBHOOK_UNAUTHORIZED`
- Webhook duplicate idempotency → prior ACK 200
- Manual deliver duplicate `Idempotency-Key` → 409 `IDEMPOTENCY_REPLAY`
- DLQ replay `SAME_IDEMPOTENCY` \| `NEW_IDEMPOTENCY` writes `integration_dlq_replay` audit
- Seed connector `rest.json.v1` (OAUTH2_CC + capabilities) and `webhook.in.v1`
- Outbox stream `jesloterp:integration:outbox`
- REST adapter stubbed — records attempt, no real GST/partner HTTP

## 10. Validation, permissions & errors

API §0 codes as `exception.code`. Permissions from GUIDE §6 / API §11 via `require_integration_permission`.

## 11. Integrations

Depends on `p13_event_bus`, `p14_messaging`, `p22_api` (module graph). Domain events go to outbox `jesloterp:integration:outbox`. p14 job ids allocated in-store (no live worker call). p19 audit represented by DLQ replay audit records.

## 12. Tests

Module + API families + durable SoR. **22 passed**.

Covered: 65 tables; deps `[p13, p14, p22]`; load after p22; outbox stream; secret stripped from GET connection; circuit open 423; mapping test pass+fail; webhook 401 vs 200; DLQ replay; deliver idempotency; persist-then-fetch outbound/attempts; empty `[]`; adapter ports.

## 13. Test execution results

`python -m pytest platforms/p23_integration/tests -q --tb=short` → **22 passed**.

## 14. RTM

`INTEGRATION_RTM.md`.

## 15. Issues found & resolved

`{**x or {}, **y}` is a SyntaxError on this Python (unpack/`or` precedence); rewritten with an explicit merge. `IntegrationAuthMode.SIGNED_JWT` attribute access failed under the test runner; seed uses string literals. Connection create now copies `*_ref` first so plaintext conversion cannot overwrite a provided vault path.

## 16. Regression

p23 suite green. Full-repo suite not run (TASK-014).

## 17. Coverage & completion

TASK-SOR-021 acceptance bar met: implement + tests + GUIDE/SCHEMA/API + RTM + this record. Not Production.

## 18. Limitations

1. Outbound + attempts persist on Postgres. Empty list is `[]`. `require_integration_access` sets RLS GUCs. Connector catalog, mappings, pipelines, DLQ, inbound receptions still memory. Not Production.
2. Adapters stay ports: REST stub never calls GST/partners; unknown kinds are `PROVIDER_PENDING`. No invented SAP/Salesforce clients.
3. Alembic not applied to live DB in-session; rate/circuit counters are process-local; webhook HMAC verify keeps current plaintext only in process memory (`_secret_plain`), never returned on GET.
