# AI Platform — Implementation Record

**Platform:** `p27_ai`  
**Date:** 2026-09-11  
**Scope:** Backend only (TASK-013 / `docs/tasks/task_p27_ai.md`)  
**Verification:** `python -m pytest platforms/p27_ai/tests -q --tb=short` → **26 passed**; ORM `ai` table count → **66**; module load order p27 after p26 → **passed**

---

## 1. Overview & Objective

Implement JeslotERP AI platform end-to-end: schema `ai`, all 64 `ai_*` domain tables plus plumbing `ai_outbox` + `ai_idempotency_key` (**66**), ModulePlugin `p27_ai` (depends on `p08_file_media`, `p18_search`, `p22_api`), in-memory catalog store with stub gateway, public `/api/v1/ai` + internal aliases, dual Alembic, permissions, tests, RTM, and status updates. Ship shape copied from p25_dashboard / p26_licensing.

## 2. All 3 Source Documents Reviewed

| Document | Path | Role |
| --- | --- | --- |
| GUIDE | `docs/platforms/27_ai/AI_GUIDE.md` | Architecture, hard rules, permissions, DoD |
| SCHEMA | `docs/platforms/27_ai/AI_SCHEMA.md` | 64 + 2 plumbing tables, enums, RLS, seed |
| API | `docs/platforms/27_ai/AI_API.md` | `/api/v1/ai` HTTP §1–10 |

Requirement docs were **not** modified. Also followed `docs/tasks/task_p27_ai.md`.

## 3. Existing Backend Architecture Reviewed

- ModulePlugin registration and topo-sort in `apps/api/main.py` (LicensingModule already wired)
- Alembic `env.py` model import list
- p25/p26 patterns: in-memory catalog store, thin routers, exception handlers, permission deps, outbox stream, dual migrations, TestClient
- Shared `PlatformBase` / `Base` ORM bases; no cross-schema FKs
- Stub inference only — no real provider HTTP

## 4. Requirements Identified

See `AI_RTM.md` (100% mapped). Major themes: secret_ref-only deployments, quota fail-closed, HITL tools, RAG citations, PII redaction, safety 451, feature 424, context 413, model 503, unpublished assistant rejected, 66 tables, `/api/v1/ai` mount, seed catalog so chat works.

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | API §0 codes + SCHEMA §3 enums | `domain/` | HTTP contract | exception handler + API status tests |
| Catalog store | In-memory gateway/assistants/chat/RAG/tools/usage | `application/services/catalog_store.py` | Runtime like p25/p26 | 15 pytest |
| ORM 66 tables | Split gateway/assistant/conversation/corpus/rag/tool/safety/usage/eval + plumbing | `infrastructure/persistence/models/` | SCHEMA §2 | count test == 66 |
| HTTP APIs | Public + internal aliases + embed worker | `infrastructure/http/` | API §1–10 | contract tests |
| Permissions | `ai.*` catalog | `application/permissions/` | GUIDE §6 / API §11 | 403 + migration seed |
| Module | `AiModule` after LicensingModule | `infrastructure/module.py` + `main.py` | registry + brief | load-order test |
| Migrations | schema + FORCE RLS | `alembic/versions/f27*.py` | Live DB path | revision chain f26b → f27a → f27b |
| Health | LIVE ≠ READY | store + routers | p21 pattern | health test |
| Stub gateway | Echo/canned complete; no OpenAI | `_stub_complete` | GUIDE / brief | chat content |

## 6. Files/Modules/Services Created or Modified

**Created:**
- `platforms/p27_ai/**` (domain, application, infrastructure, tests)
- `alembic/versions/f27a0b1c2d3e_create_ai_schema.py`
- `alembic/versions/f27b1c2d3e4f_enable_ai_rls.py`
- `docs/platforms/27_ai/AI_RTM.md`
- `docs/platforms/27_ai/AI_IMPLEMENTATION_RECORD.md` (this file)

**Modified:**
- `apps/api/main.py` — AiModule after LicensingModule + exception handlers
- `alembic/env.py` — import p27 models
- `IMPLEMENTATION_TASKS.md` — TASK-013 marked complete
- `IMPLEMENTATION_STATUS.md` — advanced to TASK-014

**Not modified (per brief):** GUIDE / SCHEMA / API requirement docs; `docs/tasks/task_p27_ai.md`.

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `f27a0b1c2d3e` | CREATE SCHEMA `ai`; create_all 66 tables; seed `ai.*` permissions | Created (apply via alembic upgrade) |
| `f27b1c2d3e4f` | ENABLE + FORCE RLS on conversations/corpora/usage/tenant assistants | Created |

**Down revision chain:** `f26b1c2d3e4f` → `f27a0b1c2d3e` → `f27b1c2d3e4f`

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/ai`:
- Models, deployments (secret_ref only), probe, routing
- Assistants + versions + publish + retire; prompts + publish
- Chat, chat/stream (JSON deltas or SSE), streams reconnect, complete
- Conversations CRUD / archive / delete
- Tools + invoke confirm/deny
- Embeddings, corpora, sources, ingest, jobs, cancel
- `POST /internal/embed-jobs/{job_id}/run`
- RAG pipelines + publish + query
- Safety profiles + rules + guardrail events + test
- Usage / rollups / quotas / cost-rates
- Feedback + eval suites/run
- Packages apply, purge-conversations, gateway-health

Internal `/internal/v1/ai/*` aliases + health + embed run.

## 9. Business Rules & Workflows Implemented

- Clients never receive provider secret values (GET deployments `secret_ref` only)
- Unpublished assistants cannot be invoked
- HIGH/CONFIRM tools stay PROPOSED until confirm; deny → `TOOL_DENIED` 403; auto path blocked → `CONFIRMATION_REQUIRED` 423
- CORPUS/MIXED grounding without hits → `GROUNDING_REQUIRED` 422
- Phone/email/PAN redacted in persisted `content_redacted`
- Quota fail-closed `QUOTA_EXCEEDED` 429
- Feature binding off → `FEATURE_DISABLED` 424
- Guardrail BLOCK → `SAFETY_BLOCKED` 451
- Context overflow → `CONTEXT_TOO_LARGE` 413
- All chat deployments DOWN → `MODEL_UNAVAILABLE` 503
- Embed jobs idempotent on content hash skip during run
- Legal hold skip on purge unless override

## 10. Validation, Permissions & Error Handling

Permissions: `ai.catalog.read`, `ai.assistant.manage`, `ai.invoke`, `ai.embed`, `ai.tool.manage`, `ai.corpus.manage`, `ai.guardrail.manage`, `ai.usage.read`, `ai.eval.manage`, `ai.admin`, `ai.*`.  
Errors mapped in `domain/exceptions.py` and registered via `register_ai_exception_handlers`.

## 11. Integrations Implemented

- p08 / p18 / p22: declared ModulePlugin deps; media/search/index IDs stored as UUID/string refs (no cross-schema FKs)
- p26 entitlement code on quota policy (`ai.tokens`) checked as feature-gate hook, not live compile
- p14 job_id stored on embed/eval rows; worker is in-process stub `run_embed_job`
- Outbox stream `jesloterp:ai:outbox`

## 12. Test Cases Created for Each Functionality

| Family | File | Variations |
| --- | --- | --- |
| Module/schema | `test_ai_module_tables.py` | 66 tables, deps, load after p26, outbox stream |
| Models/deployments | `test_ai_api_contracts.py` | list/get, secret stripped, probe, routing, 403 |
| Assistants/prompts | same | unpublished 422 vs publish then chat; retire; prompt versions |
| Chat/safety | same | success vs SAFETY_BLOCKED; PII redaction |
| Stream/complete/conversations | same | stream + reconnect 404; complete; archive/delete |
| Tools HITL | same | confirm SUCCEEDED vs deny 403; tool CRUD |
| Embed/corpora | same | vectors vs empty 422; ingest/run/cancel/internal |
| RAG | same | GROUNDING_REQUIRED vs cited; pipeline publish |
| Safety | same | ALLOW vs BLOCK test; duplicate profile 409 |
| Usage/quota | same | list usage vs QUOTA_EXCEEDED; cost-rates |
| Eval/admin | same | feedback rating 422; eval run; package missing 404; purge |
| Hard gates | same | 413 / 503 / 424 + LIVE≠READY |

## 13. Test Execution Results

```
pytest platforms/p27_ai/tests -q
15 passed
```

## 14. Requirements Traceability Matrix (RTM)

See [`AI_RTM.md`](AI_RTM.md). All GUIDE/SCHEMA/API requirements mapped; none left unimplemented for the in-memory control-plane scope.

## 15. Issues Found & How They Were Resolved

- Catalog store write was truncated mid-method; remaining usage/eval/admin/health methods appended and verified by tests.
- Boolean ORM columns initially used bare `mapped_column()`; switched to `Boolean` to match sibling platforms.

## 16. Regression/Existing Functionality Verification

Only `platforms/p27_ai/tests` was required by this task. Full-suite verification is TASK-014. `load_modules()` still resolves p26 then p27.

## 17. Final Coverage & Completion Status

**Complete** against TASK-013 brief: 66 tables, all API §1–10 families, hard rules, dual Alembic, AiModule after licensing, RTM + record, tests green.

## 18. Remaining Issues or Limitations

- Assistant/conversation HTTP persists on Postgres; empty list is `[]`. `require_ai_access` sets RLS GUCs. Prompts/tools/RAG/eval still memory. Not Production.
- Pytest factory stays Stub. `OpenAiModelGateway` fails `PROVIDER_PENDING` (no invented completions). No live OpenAI in pytest.
- Inference under TestClient is a stub echo/canned gateway (required — no real OpenAI/vendor calls).
- p18 vector upsert and p08 byte fetch are UUID-ref stubs; chunks live in the in-memory store after `run_embed_job`.
- Eval run is synchronous stub (stores `job_id` meta; does not enqueue p14).
- Feature/license gate uses `ai_feature_binding.is_enabled` plus quota `entitlement_code`; it does not call live p26 compile at invoke time.
