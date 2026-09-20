# AI Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p27_ai/tests -q --tb=short` → **26 passed**; **66** `ai_*` tables (64 domain + `ai_outbox` + `ai_idempotency_key`); load order after p26; outbox `jesloterp:ai:outbox`.

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| AI-G-01 | GUIDE §1 | AI control plane: models, assistants, prompts, conversations, embeddings/RAG, tools, guardrails, usage, evals, packs | `AiCatalogStore` + schema `ai` | Implemented | module tables + health | PASS |
| AI-G-02 | GUIDE §1 | Does not own p11 rules, p18 engine, p08 bytes, p22 product keys, p26 entitlements | UUID refs only; stub gateway | Implemented | no cross-schema FK tables | PASS |
| AI-G-03 | GUIDE §2 | No browser→vendor API keys; all inference through p27 gateway | `/chat` `/complete` `/embeddings` stub | Implemented | chat + embeddings | PASS |
| AI-G-04 | GUIDE §2 | Tool calls need permission + HITL | `_invoke_tool` confirm/deny | Implemented | HITL confirm vs deny | PASS |
| AI-G-05 | GUIDE §2 | RAG only from authorized corpora; citations when policy requires | `_retrieve` + `require_citations` | Implemented | grounding required vs cited | PASS |
| AI-G-06 | GUIDE §2 | Redact PII in persisted logs | `redact_pii` on messages | Implemented | chat PII redaction | PASS |
| AI-G-07 | GUIDE §2 | Quotas fail-closed | `_assert_quota` → 429 | Implemented | quota exceeded | PASS |
| AI-G-08 | GUIDE §2 | No cross-schema FKs | UUID columns on ORM | Implemented | table inventory | PASS |
| AI-G-09 | GUIDE §2 | Feature/license gates before expensive models | `ai_feature_binding` → 424 | Implemented | FEATURE_DISABLED | PASS |
| AI-G-10 | GUIDE §6 | Permissions `ai.catalog.read` … `ai.*` | `require_ai_permission` | Implemented | catalog 403 | PASS |
| AI-G-11 | GUIDE §6 | FORCE RLS conversations/corpora/usage/tenant assistants | Alembic `f27b1c2d3e4f` | Implemented | migration present | PASS |
| AI-G-12 | GUIDE §8 | Outbox events + stream | `_emit` + `jesloterp:ai:outbox` | Implemented | outbox stream test | PASS |
| AI-G-13 | GUIDE §10 | Stub gateway; do not call real OpenAI | `_stub_complete` echo | Implemented | chat stub content | PASS |
| AI-S-01 | SCHEMA §1 | Schema `ai` (never p27); tables `ai_*` | `AI_SCHEMA` | Implemented | table names | PASS |
| AI-S-02 | SCHEMA §2 | 64 listed tables + plumbing = 66 | ORM models | Implemented | `test_ai_module_tables_match_schema_list` | PASS |
| AI-S-03 | SCHEMA §3 | Enums modality/grounding/role/status/risk/confirm/guardrail/embed/lifecycle | `domain/enums.py` | Implemented | API payloads | PASS |
| AI-S-04 | SCHEMA §4–10 | Model/deployment/assistant/conversation/corpus/tool/safety/usage/eval columns | split model modules | Implemented | table inventory | PASS |
| AI-S-05 | SCHEMA §11 | `ai_outbox` + `ai_idempotency_key` | outbox + idempotency models | Implemented | table names | PASS |
| AI-S-06 | SCHEMA §12 | FORCE RLS tenant assistants/corpora/conversations/usage | Alembic FORCE list | Implemented | migration | PASS |
| AI-S-07 | SCHEMA §13 + brief | Seed chat+embed models, deployment secret_ref, published assistant, safety, quota, package | `seed_defaults` | Implemented | catalog/chat/packages | PASS |
| AI-A-00 | API §0 | Envelope + FORBIDDEN/TOOL_DENIED/NOT_FOUND/CONFLICT/CONTEXT_TOO_LARGE/VALIDATION/GROUNDING_REQUIRED/CONFIRMATION_REQUIRED/FEATURE_DISABLED/QUOTA_EXCEEDED/RATE_LIMITED/MODEL_UNAVAILABLE/SAFETY_BLOCKED | `domain/exceptions.py` | Implemented | dedicated hard-rule tests | PASS |
| AI-A-01 | API §1 | Models/deployments/probe/routing; GET strips secrets | `/models` `/deployments*` `/routing-policies` | Implemented | models family | PASS |
| AI-A-02 | API §2.1 | Assistants CRUD + versions + publish + retire | `/assistants*` | Implemented | assistants family | PASS |
| AI-A-03 | API §2.2 | Prompts + versions + publish | `/prompts*` | Implemented | prompts in assistants test | PASS |
| AI-A-04 | API §3.1–3.3 | Chat / stream / complete; unpublished rejected | `/chat` `/chat/stream` `/complete` `/streams/{id}` | Implemented | chat + stream + unpublished | PASS |
| AI-A-05 | API §3.4 | Conversations list/get/messages/archive/delete | `/conversations*` | Implemented | conversations CRUD | PASS |
| AI-A-06 | API §4 | Tools registry + invoke confirm/deny | `/tools*` `/tool-invocations*` | Implemented | HITL family | PASS |
| AI-A-07 | API §5 | Embeddings, corpora, sources, ingest, jobs, cancel, internal run | `/embeddings` `/corpora*` `/embed-jobs*` `/internal/embed-jobs/{id}/run` | Implemented | embeddings/corpora family | PASS |
| AI-A-08 | API §6 | RAG pipelines + publish + query | `/rag/*` | Implemented | RAG family | PASS |
| AI-A-09 | API §7 | Safety profiles + rules + events + test | `/safety-profiles*` `/guardrail*` | Implemented | safety family | PASS |
| AI-A-10 | API §8 | Usage/rollups/quotas/cost-rates | `/usage*` `/quotas` `/cost-rates` | Implemented | usage family | PASS |
| AI-A-11 | API §9 | Feedback + eval suites/run | `/feedback` `/eval/*` | Implemented | feedback/eval family | PASS |
| AI-A-12 | API §10 | Packages apply, purge-conversations, gateway-health | `/packages*` `/admin/*` | Implemented | admin family | PASS |
| AI-A-13 | API + brief | Internal `/internal/v1/ai/*` aliases + health | `api_v1.py` | Implemented | internal health + embed run | PASS |
| AI-A-14 | API §0 / GUIDE hard rules | Secret stripped; SAFETY_BLOCKED; HITL; quota 429; grounding; unpublished | store guards | Implemented | dedicated tests | PASS |
| AI-M-01 | brief | Package `platforms.p27_ai`; ModulePlugin after p26; deps p08/p18/p22 | `AiModule` + `main.py` | Implemented | module plugin + load order | PASS |
| AI-M-02 | brief | Dual Alembic f27a revises f26b; f27b FORCE RLS | `alembic/versions/f27*.py` | Implemented | revision chain | PASS |
| AI-SOR-01 | TASK-SOR-022 | Assistant/conversation HTTP → Postgres; empty `[]` | `catalog_repository` + `require_ai_access` | Implemented | persist-then-fetch + db-first empty | PASS |
| AI-SOR-02 | TASK-SOR-022 | Model gateway port; no fake OpenAI | `OpenAiModelGateway` `PROVIDER_PENDING` | Implemented | pytest stub factory + live class fail-closed | PASS |
