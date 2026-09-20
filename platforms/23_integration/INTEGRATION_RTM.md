# Integration Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p23_integration/tests -q --tb=short` → **22 passed**; 65 `integration` tables; load order after p22.

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| INT-G-01 | GUIDE §1 | External connectivity control plane (connectors, connections, mappings, pipelines, inbound/outbound) | `IntegrationCatalogStore` + schema `integration` | Implemented | module tables + health | PASS |
| INT-G-02 | GUIDE §1 | Not ad-hoc HTTP from domains; pipelines + adapters | stub REST adapter + pipeline deliver | Implemented | outbound stub attempt | PASS |
| INT-G-03 | GUIDE §2 | No plaintext secrets in tables/API | secret_ref / vault_path; `_strip_secrets` | Implemented | GET connection strips secret | PASS |
| INT-G-04 | GUIDE §2 | Outbound via pipelines; inbound fail-closed on bad signature | deliver + webhook HMAC | Implemented | webhook 401 vs 200 | PASS |
| INT-G-05 | GUIDE §2 | At-least-once + idempotency inbound & outbound | deliver Idempotency-Key; webhook replay ACK | Implemented | deliver 409; webhook 200 replay | PASS |
| INT-G-06 | GUIDE §2 | Heavy I/O via p14; p23 stores control + attempts | job_id on outbound; stub deliver | Implemented | deliver PENDING + job_id | PASS |
| INT-G-07 | GUIDE §2 | No cross-schema FKs | UUID refs only on ORM | Implemented | table inventory | PASS |
| INT-G-08 | GUIDE §2 | Circuit open stops new attempts | `CircuitOpenError` 423 | Implemented | circuit open 423 | PASS |
| INT-G-09 | GUIDE §4 | Connector ≠ connection; mapping; pipeline; delivery; reception; DLQ; partner profile | store entities + APIs | Implemented | all API families | PASS |
| INT-G-10 | GUIDE §6 | Permissions `integration.*` | `require_integration_permission` | Implemented | 403 catalog | PASS |
| INT-G-11 | GUIDE §6 | FORCE RLS tenant connections/deliveries/receptions/DLQ/profiles | Alembic `f23b1c2d3e4f` | Implemented | migration present | PASS |
| INT-G-12 | GUIDE §8 | Domain events connection/pipeline/delivery/inbound/dlq/circuit/mapping | outbox `_emit` | Implemented | store emit + stream test | PASS |
| INT-G-13 | GUIDE §10 | DoD: no domain HTTP, secrets, webhook 401, DLQ replay, circuit, inbound idempotent, tenant isolation, no XFKs | store + APIs | Implemented | secrets/circuit/webhook/DLQ | PASS |
| INT-S-01 | SCHEMA §1 | Schema `integration` (never p23); `integration_*` tables | `INTEGRATION_SCHEMA` | Implemented | table names | PASS |
| INT-S-02 | SCHEMA §2 | 63 domain + outbox + idempotency = 65 | ORM models | Implemented | `test_integration_module_tables_count_65` | PASS |
| INT-S-03 | SCHEMA §3 | Enums kind/auth/env/direction/step/status/error/breaker/lifecycle | `domain/enums.py` | Implemented | API payloads | PASS |
| INT-S-04 | SCHEMA §4 | Connector + connection + auth secret refs | catalog + connection models/store | Implemented | connectors + connections | PASS |
| INT-S-05 | SCHEMA §5–6 | Mapping versions/fields; pipeline steps/triggers | mapping + pipeline APIs | Implemented | mapping + pipeline families | PASS |
| INT-S-06 | SCHEMA §7 | Outbound message, attempts, retry, rate, circuit, bulkhead, payload/response | outbound models + APIs | Implemented | outbound + retry/rate | PASS |
| INT-S-07 | SCHEMA §8 | Webhook endpoint + inbound reception unique (endpoint, key) | inbound models + hooks | Implemented | webhook family | PASS |
| INT-S-08 | SCHEMA §9 | DLQ + replay audit; reconcile unique; sync_state | dlq + reconcile APIs | Implemented | DLQ + reconcile | PASS |
| INT-S-09 | SCHEMA §10 | Partner profiles; seed packs sample.rest / webhook / gst / payments / fleet | seed + partner APIs | Implemented | partners + packages | PASS |
| INT-S-10 | SCHEMA §11 | `integration_outbox`, `integration_idempotency_key` | outbox + idempotency models | Implemented | table names + stream | PASS |
| INT-S-11 | SCHEMA §13 | Seed rest.json.v1, webhook.in.v1, default.exp5, breaker 5/60s, InvoicePosted/PaymentCaptured, perms, core.samples.v1 | `seed_defaults` + Alembic perms | Implemented | connectors + retry + packages | PASS |
| INT-A-00 | API §0 | Envelope + errors AUTH/WEBHOOK_UNAUTHORIZED/FORBIDDEN/NOT_FOUND/CONFLICT/IDEMPOTENCY_REPLAY/VALIDATION/MAPPING_FAILED/CIRCUIT_OPEN/RATE_LIMITED/DEPENDENCY_UNAVAILABLE | `domain/exceptions.py` | Implemented | all API families | PASS |
| INT-A-01 | API §1 | Connector catalog list/get/config-schema | `/connectors*` | Implemented | connectors family | PASS |
| INT-A-02 | API §2 | Connections CRUD/disable/enable/probe/rotate/health/circuit | `/connections*` | Implemented | connections family | PASS |
| INT-A-03 | API §3 | Mappings + versions + publish + test | `/mappings*` | Implemented | mapping pass+fail | PASS |
| INT-A-04 | API §4 | Pipelines define/publish/retire + manual deliver + Idempotency-Key | `/pipelines*` | Implemented | deliver + retire | PASS |
| INT-A-05 | API §5 | Outbound list/get/attempts/cancel/payload; internal deliver/complete | `/outbound-messages*` `/internal/deliver*` | Implemented | outbound + aliases | PASS |
| INT-A-06 | API §6 | Retry policies + rate-limit policies + PUT connection rate-limit | `/retry-policies` `/rate-limit-policies` | Implemented | retry + 429 | PASS |
| INT-A-07 | API §7 | DLQ get/replay/discard/bulk; SAME_IDEMPOTENCY \| NEW_IDEMPOTENCY + audit | `/dlq*` | Implemented | DLQ replay/discard | PASS |
| INT-A-08 | API §8 | Webhook admin + public `/api/v1/hooks/{slug}/{key}` no JWT; GET challenge; inbound complete | `/webhooks*` `/hooks*` | Implemented | 401 vs 200 + GET challenge | PASS |
| INT-A-09 | API §9 | Partner profiles + reconcile + sync-state reset | `/partner-profiles*` `/reconcile*` `/sync-state*` | Implemented | partners family | PASS |
| INT-A-10 | API §10 | Analytics usage/errors; packages apply; alert-rules | `/analytics*` `/packages*` `/alert-rules*` | Implemented | analytics/packs/alerts | PASS |
| INT-A-11 | API §11 | Permission matrix + internal service principal | HTTP gates + `/internal/v1/integration/*` | Implemented | 403 + aliases | PASS |
| INT-MOD | registry / brief | ModulePlugin after p22; deps p13+p14+p22; Alembic f23a/f23b; outbox `jesloterp:integration:outbox` | `IntegrationModule` + main/env | Implemented | load order + deps + stream | PASS |
| INT-SOR-01 | TASK-SOR-021 | Outbound + attempts → Postgres; empty `[]` | `outbound_repository` + `require_integration_access` | Implemented | persist-then-fetch + db-first empty | PASS |
| INT-SOR-02 | TASK-SOR-021 | Connector adapters stay ports; no fake GST/SAP | `StubRestAdapter` + `PendingConnectorAdapter` | Implemented | stub adapter + PROVIDER_PENDING | PASS |
