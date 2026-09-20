# JeslotERP AI Platform — Complete API Endpoints

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — assistants/conversations list+create/chat persist Postgres-first; empty list is `[]`. OpenAI is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p27_ai`  
**Base path:** `/api/v1/ai`  
**Companion:** [`AI_GUIDE.md`](AI_GUIDE.md) · [`AI_SCHEMA.md`](AI_SCHEMA.md)

> All inference goes through this control plane (registered in p22). Clients never hold provider API keys.

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes | Bearer JWT |
| `X-Tenant-Id` | Yes | Tenant scope |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Invoke / embed jobs | Recommended |
| `Accept` | Streaming | `text/event-stream` for chat stream |

### Envelope

Non-stream responses use `StandardResponse`.  
Streaming: SSE events `message.delta`, `tool.proposed`, `citation`, `message.completed`, `error`.

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 403 | `FORBIDDEN` / `TOOL_DENIED` | AuthZ / tool policy |
| 404 | `NOT_FOUND` | Unknown assistant/thread |
| 409 | `CONFLICT` | Duplicate idempotency |
| 413 | `CONTEXT_TOO_LARGE` | Prompt+RAG overflow |
| 422 | `VALIDATION_ERROR` / `GROUNDING_REQUIRED` | Bad args / missing cites |
| 423 | `CONFIRMATION_REQUIRED` | HITL tool pending |
| 424 | `FEATURE_DISABLED` | Flag/license |
| 429 | `QUOTA_EXCEEDED` / `RATE_LIMITED` | Tokens or RPS |
| 503 | `MODEL_UNAVAILABLE` | Deployments down |
| 451 | `SAFETY_BLOCKED` | Guardrail block |

---

## 1. Model catalog & deployments

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/models` | `ai.catalog.read` |
| `GET` | `/api/v1/ai/models/{model_key}` | `ai.catalog.read` |
| `GET` | `/api/v1/ai/deployments` | `ai.admin` |
| `POST` | `/api/v1/ai/deployments` | `ai.admin` |
| `PATCH` | `/api/v1/ai/deployments/{id}` | `ai.admin` |
| `POST` | `/api/v1/ai/deployments/{id}/probe` | `ai.admin` |
| `PUT` | `/api/v1/ai/routing-policies` | `ai.admin` |

**POST deployment:** `provider`, `model_key`, `base_url`, `secret_ref`, `region`, `priority` — secret values never returned.

---

## 2. Assistants & prompts

### 2.1 Assistants

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/assistants` | `ai.catalog.read` |
| `POST` | `/api/v1/ai/assistants` | `ai.assistant.manage` |
| `GET` | `/api/v1/ai/assistants/{assistant_id}` | `ai.catalog.read` |
| `PATCH` | `/api/v1/ai/assistants/{assistant_id}` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/assistants/{assistant_id}/versions` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/assistants/{assistant_id}/publish` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/assistants/{assistant_id}/retire` | `ai.assistant.manage` |

**POST create:**

```json
{
  "assistant_key": "ops.help",
  "name": "Ops Copilot",
  "grounding_mode": "CORPUS",
  "default_model_key": "chat.default",
  "safety_profile_key": "standard",
  "corpus_ids": ["…"],
  "tool_keys": ["trips.search", "navigation.help"],
  "prompt": {
    "template_key": "ops.help.system",
    "body": "You are JeslotERP ops assistant. Cite sources. Do not invent rates."
  }
}
```

### 2.2 Prompts

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/prompts` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/prompts` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/prompts/{id}/versions` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/prompts/{id}/publish` | `ai.assistant.manage` |

---

## 3. Runtime — chat / complete / stream

### 3.1 Start or continue conversation

`POST /api/v1/ai/chat`  
**Permission:** `ai.invoke`  
**Header:** `Idempotency-Key` recommended  

```json
{
  "assistant_id": "…",
  "conversation_id": null,
  "message": "Show open trips for HO branch today",
  "filters": { "branch_id": "…" },
  "stream": false,
  "options": { "temperature": 0.2, "max_output_tokens": 800 }
}
```

**Response (non-stream):**

```json
{
  "conversation_id": "…",
  "message_id": "…",
  "role": "ASSISTANT",
  "content": "There are 42 open trips…",
  "citations": [
    { "title": "Trip policy", "media_id": "…", "snippet": "…", "score": 0.82 }
  ],
  "tool_invocations": [
    { "id": "…", "tool_key": "trips.search", "status": "SUCCEEDED", "result_summary": "42 rows" }
  ],
  "usage": { "token_input": 1200, "token_output": 180 },
  "model_key": "chat.default"
}
```

### 3.2 Stream

`POST /api/v1/ai/chat/stream` — same body with SSE.  
`GET /api/v1/ai/streams/{session_id}` — reconnect support.

### 3.3 Complete (raw)

`POST /api/v1/ai/complete`  
**Permission:** `ai.invoke`  
Lower-level prompt+messages without assistant packaging (admin/tools only; still guardrailed + metered).

### 3.4 Conversations CRUD

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/conversations` | `ai.invoke` |
| `GET` | `/api/v1/ai/conversations/{id}` | `ai.invoke` |
| `GET` | `/api/v1/ai/conversations/{id}/messages` | `ai.invoke` |
| `POST` | `/api/v1/ai/conversations/{id}/archive` | `ai.invoke` |
| `DELETE` | `/api/v1/ai/conversations/{id}` | `ai.invoke` |

List is RLS-scoped to caller (unless admin).

---

## 4. Tools (HITL)

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/tools` | `ai.catalog.read` |
| `POST` | `/api/v1/ai/tools` | `ai.tool.manage` |
| `PATCH` | `/api/v1/ai/tools/{tool_id}` | `ai.tool.manage` |
| `POST` | `/api/v1/ai/tools/{tool_id}/versions` | `ai.tool.manage` |
| `GET` | `/api/v1/ai/tool-invocations/{id}` | `ai.invoke` |
| `POST` | `/api/v1/ai/tool-invocations/{id}/confirm` | `ai.invoke` |
| `POST` | `/api/v1/ai/tool-invocations/{id}/deny` | `ai.invoke` |

**Confirm body:** `{ "reason": "Approved rate suggestion apply" }`  
Server re-checks permissions and risk policy, then executes handler.

If chat response includes `PROPOSED` tools, client must confirm before side effects. Status `423 CONFIRMATION_REQUIRED` when an auto path is blocked.

---

## 5. Embeddings & corpora

### 5.1 Embed

`POST /api/v1/ai/embeddings`  
**Permission:** `ai.embed`  

```json
{
  "model_key": "embed.default",
  "inputs": ["text one", "text two"]
}
```

**Response:** vectors **or** vector ids depending on policy (often ids only for large payloads). Metered as embed units.

### 5.2 Corpora

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/corpora` | `ai.corpus.manage` |
| `POST` | `/api/v1/ai/corpora` | `ai.corpus.manage` |
| `POST` | `/api/v1/ai/corpora/{id}/sources` | `ai.corpus.manage` |
| `POST` | `/api/v1/ai/corpora/{id}/ingest` | `ai.corpus.manage` |
| `GET` | `/api/v1/ai/corpora/{id}/jobs` | `ai.corpus.manage` |
| `GET` | `/api/v1/ai/embed-jobs/{job_id}` | `ai.corpus.manage` |
| `POST` | `/api/v1/ai/embed-jobs/{job_id}/cancel` | `ai.corpus.manage` |

**Add sources:** `{ "media_ids": ["…"], "entity_refs": [{ "type": "Document", "id": "…" }] }`  
**Ingest:** enqueues p14 job; chunks → embed → upsert p18 vector index.

### 5.3 Internal worker

`POST /api/v1/ai/internal/embed-jobs/{job_id}/run` — service principal.

---

## 6. RAG

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/rag/pipelines` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/rag/pipelines` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/rag/pipelines/{id}/publish` | `ai.assistant.manage` |
| `POST` | `/api/v1/ai/rag/query` | `ai.invoke` |

**RAG query (debug/retrieve-only):**

```json
{
  "corpus_id": "…",
  "query": "detention charges policy",
  "top_k": 8
}
```

**Response:** hits with scores/snippets — no generation. Full grounded chat uses `/chat` with assistant grounding.

---

## 7. Guardrails & safety

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/safety-profiles` | `ai.guardrail.manage` |
| `POST` | `/api/v1/ai/safety-profiles` | `ai.guardrail.manage` |
| `PUT` | `/api/v1/ai/safety-profiles/{id}/rules` | `ai.guardrail.manage` |
| `GET` | `/api/v1/ai/guardrail-events` | `ai.guardrail.manage` |
| `POST` | `/api/v1/ai/guardrails/test` | `ai.guardrail.manage` |

**Test:** `{ "profile_key": "regulated", "text": "…" }` → `{ "action": "REDACT"|"BLOCK"|"ALLOW", "hits": [] }`

---

## 8. Usage, quotas, entitlements

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/usage` | `ai.usage.read` |
| `GET` | `/api/v1/ai/usage/rollups` | `ai.usage.read` |
| `GET` | `/api/v1/ai/quotas` | `ai.usage.read` |
| `PUT` | `/api/v1/ai/quotas` | `ai.admin` |
| `GET` | `/api/v1/ai/cost-rates` | `ai.admin` |
| `PUT` | `/api/v1/ai/cost-rates` | `ai.admin` |

**Usage query:** `from`, `to`, `assistant_id`, `user_id`, `granularity`  
Quota upsert may reference `entitlement_code` checked against p26 at invoke time.

---

## 9. Feedback & evaluations

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/ai/feedback` | `ai.invoke` |
| `GET` | `/api/v1/ai/feedback` | `ai.eval.manage` |
| `GET` | `/api/v1/ai/eval/suites` | `ai.eval.manage` |
| `POST` | `/api/v1/ai/eval/suites` | `ai.eval.manage` |
| `POST` | `/api/v1/ai/eval/suites/{id}/run` | `ai.eval.manage` |
| `GET` | `/api/v1/ai/eval/runs/{run_id}` | `ai.eval.manage` |

**Feedback:** `{ "message_id": "…", "rating": 1, "comment": "Wrong branch" }` (`rating` -1|0|1)

**Eval run** async via p14; blocks assistant promote when gate configured and score < threshold.

---

## 10. Packs & admin

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/ai/packages` | `ai.admin` |
| `POST` | `/api/v1/ai/packages/{package_key}/apply` | `ai.admin` |
| `POST` | `/api/v1/ai/admin/purge-conversations` | `ai.admin` |
| `GET` | `/api/v1/ai/admin/gateway-health` | `ai.admin` |

**Purge:** retention policy; never purge under legal hold without override permission.

---

## 11. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Catalog models/assistants/tools | `ai.catalog.read` |
| Chat / conversations / feedback | `ai.invoke` |
| Embeddings API | `ai.embed` |
| Assistants/prompts | `ai.assistant.manage` |
| Corpora/ingest | `ai.corpus.manage` |
| Tool registry | `ai.tool.manage` |
| Guardrails | `ai.guardrail.manage` |
| Usage read | `ai.usage.read` |
| Evals | `ai.eval.manage` |
| Deployments/quotas/packs | `ai.admin` |

---

## 12. Example flows

### 12.1 Grounded ops question

1. User opens Ops Copilot (`grounding_mode=CORPUS`)  
2. `POST /chat` with question  
3. Gateway: quota → guardrail → RAG retrieve (p18) → generate → citations  
4. Optional LOW tool `trips.search` AUTO  
5. Usage event recorded  

### 12.2 High-risk tool

1. Model proposes `invoices.adjust` (CRITICAL/CONFIRM)  
2. Response includes `tool_invocations[].status=PROPOSED`  
3. User `POST .../confirm`  
4. Handler runs domain command with user AuthZ  
5. p19 audit + tool result message appended  

### 12.3 Ingest policy PDFs

1. Upload PDFs via p08  
2. `POST /corpora/{id}/sources` with media_ids  
3. `POST /ingest` → embed job  
4. Assistant corpus binding picks up new chunks  

---

## 13. Related documents

- Guide: [`AI_GUIDE.md`](AI_GUIDE.md)  
- Schema: [`AI_SCHEMA.md`](AI_SCHEMA.md)  
- Search: [`../18_search/SEARCH_API.md`](../18_search/SEARCH_API.md)  
- File media: [`../08_file_media/FILE_MEDIA_API.md`](../08_file_media/FILE_MEDIA_API.md)  
- API platform: [`../22_api/API_ENDPOINTS.md`](../22_api/API_ENDPOINTS.md)  
- Rules: [`../11_rules/RULES_API.md`](../11_rules/RULES_API.md)  
- Licensing: [`../26_licensing/V2_LICENSING_API.md`](../26_licensing/V2_LICENSING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
