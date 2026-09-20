# JeslotERP AI Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `ai_assistant` + `ai_conversation` are the HTTP ledger. Model inference stays on the gateway port. Not Production.  
**Package:** `platforms.p27_ai`  
**PostgreSQL schema:** `ai`  
**Companion:** [`AI_GUIDE.md`](AI_GUIDE.md) · [`AI_API.md`](AI_API.md)

> Runtime models: `platforms/p27_ai/infrastructure/persistence/models/`.  
> Vectors live in search/vector store; PG stores control plane + conversation meta.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `ai` (never `p27`) |
| Tables | `ai_*` |
| Soft delete | Archive threads; retire assistants |
| Cross-schema | UUID refs (tenant, user, media, search index) |
| RLS | FORCE on conversations, corpora, usage, tenant assistants |
| Secrets | Provider keys via `secret_ref` only |

---

## 2. Complete table inventory (**64 tables**)

### 2.1 Models & gateway (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `ai_model` | Logical models |
| 2 | `ai_model_capability` | chat, embed, vision… |
| 3 | `ai_provider` | Vendors |
| 4 | `ai_deployment` | Physical endpoints |
| 5 | `ai_deployment_secret` | secret_ref |
| 6 | `ai_routing_policy` | Route rules |
| 7 | `ai_fallback_chain` | Fallback order |
| 8 | `ai_gateway_health` | Probe state |
| 9 | `ai_feature_binding` | Feature gates |

### 2.2 Assistants & prompts (8)

| # | Table | Purpose |
|---|---|---|
| 10 | `ai_assistant` | Assistants |
| 11 | `ai_assistant_version` | Versions |
| 12 | `ai_prompt_template` | Prompt templates |
| 13 | `ai_prompt_version` | Prompt versions |
| 14 | `ai_prompt_locale` | Locale bodies |
| 15 | `ai_assistant_tool` | Tool allowlist |
| 16 | `ai_assistant_corpus` | Grounding corpora |
| 17 | `ai_assistant_policy` | Safety/HITL binds |

### 2.3 Conversations (7)

| # | Table | Purpose |
|---|---|---|
| 18 | `ai_conversation` | Threads |
| 19 | `ai_message` | Messages |
| 20 | `ai_message_part` | Multimodal parts |
| 21 | `ai_stream_session` | Streaming sessions |
| 22 | `ai_citation` | RAG citations |
| 23 | `ai_conversation_summary` | Summaries |
| 24 | `ai_conversation_share` | Optional shares |

### 2.4 Embeddings & corpora (10)

| # | Table | Purpose |
|---|---|---|
| 25 | `ai_corpus` | Corpora |
| 26 | `ai_corpus_source` | Media/entity sources |
| 27 | `ai_document` | Ingested docs meta |
| 28 | `ai_chunk` | Chunk meta |
| 29 | `ai_embed_job` | Embed jobs |
| 30 | `ai_embed_model_bind` | Model for corpus |
| 31 | `ai_vector_index_ref` | p18 index soft ref |
| 32 | `ai_chunk_acl` | Extra ACL hints |
| 33 | `ai_ingest_policy` | Chunking/PII policies |
| 34 | `ai_reindex_schedule` | p17 binding meta |

### 2.5 RAG (5)

| # | Table | Purpose |
|---|---|---|
| 35 | `ai_rag_pipeline` | Pipelines |
| 36 | `ai_rag_pipeline_version` | Versions |
| 37 | `ai_rag_step` | Retrieve/rerank/generate |
| 38 | `ai_rag_run` | Run meta |
| 39 | `ai_rag_hit` | Retrieved hits |

### 2.6 Tools (7)

| # | Table | Purpose |
|---|---|---|
| 40 | `ai_tool` | Tool registry |
| 41 | `ai_tool_version` | Versions / schema |
| 42 | `ai_tool_permission` | Required permissions |
| 43 | `ai_tool_invocation` | Invocations |
| 44 | `ai_tool_confirmation` | HITL confirms |
| 45 | `ai_tool_result` | Results meta |
| 46 | `ai_tool_risk_class` | Risk catalog |

### 2.7 Guardrails & safety (6)

| # | Table | Purpose |
|---|---|---|
| 47 | `ai_safety_profile` | Profiles |
| 48 | `ai_guardrail_rule` | Rules |
| 49 | `ai_blocklist_pattern` | Patterns |
| 50 | `ai_redaction_policy` | PII redaction |
| 51 | `ai_guardrail_event` | Blocks/flags |
| 52 | `ai_content_filter_hit` | Filter hits |

### 2.8 Usage, quotas, cost (6)

| # | Table | Purpose |
|---|---|---|
| 53 | `ai_usage_event` | Token/embed events |
| 54 | `ai_usage_rollup` | Aggregates |
| 55 | `ai_quota_policy` | Quotas |
| 56 | `ai_quota_counter` | Counters meta |
| 57 | `ai_cost_rate` | $/1k tokens |
| 58 | `ai_entitlement_hook` | p26 bridge |

### 2.9 Eval, feedback, governance (6)

| # | Table | Purpose |
|---|---|---|
| 59 | `ai_eval_suite` | Eval suites |
| 60 | `ai_eval_case` | Golden cases |
| 61 | `ai_eval_run` | Runs |
| 62 | `ai_feedback` | User feedback |
| 63 | `ai_package` | Packs |
| 64 | `ai_package_item` | Items |

**Plumbing:** `ai_outbox`, `ai_idempotency_key`

**Implementation total with plumbing: 66 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `ai_model_modality` | `CHAT`, `COMPLETION`, `EMBED`, `VISION`, `RERANK` |
| `ai_grounding_mode` | `NONE`, `CORPUS`, `ENTITY`, `MIXED` |
| `ai_message_role` | `SYSTEM`, `USER`, `ASSISTANT`, `TOOL` |
| `ai_conversation_status` | `ACTIVE`, `COMPLETED`, `ARCHIVED`, `BLOCKED` |
| `ai_tool_risk` | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `ai_confirm_policy` | `AUTO`, `CONFIRM`, `DENY` |
| `ai_guardrail_action` | `ALLOW`, `REDACT`, `BLOCK`, `FLAG` |
| `ai_embed_job_status` | `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED` |
| `ai_lifecycle` | `DRAFT`, `PUBLISHED`, `DEPRECATED`, `RETIRED` |

---

## 4. Models & deployments

### 4.1 `ai_model`

| Column | Type | Notes |
|---|---|---|
| `model_key` | VARCHAR(80) UNIQUE | `gpt-4.1-mini` |
| `display_name` | VARCHAR(150) | |
| `modality` | VARCHAR(20) | |
| `context_window` | INT | |
| `max_output_tokens` | INT NULL | |
| `is_active` | BOOLEAN | |

### 4.2 `ai_deployment`

| Column | Type | Notes |
|---|---|---|
| `deployment_key` | VARCHAR(80) | |
| `provider_id` | UUID | |
| `model_id` | UUID | |
| `base_url` | VARCHAR(500) NULL | |
| `region` | VARCHAR(40) NULL | |
| `secret_ref` | VARCHAR(200) | |
| `status` | VARCHAR(20) | |
| `priority` | INT | Routing |

### 4.3 `ai_routing_policy`

Match on `tenant_id`, `assistant_id`, `modality` → deployment; optional fallback_chain_id.

---

## 5. Assistants & prompts

### 5.1 `ai_assistant`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system |
| `assistant_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `lifecycle` | VARCHAR(20) | |
| `grounding_mode` | VARCHAR(20) | |
| `default_model_id` | UUID NULL | |
| `safety_profile_id` | UUID NULL | |
| `published_version_id` | UUID NULL | |

### 5.2 `ai_prompt_version`

| Column | Type | Notes |
|---|---|---|
| `prompt_template_id` | UUID | |
| `version` | INT | |
| `checksum` | VARCHAR(64) | |
| `body` | TEXT | May include `{{vars}}` |
| `lifecycle` | VARCHAR(20) | |

### 5.3 `ai_assistant_version`

Binds prompt_version, tool set snapshot, corpus set, temperature/top_p defaults, rag_pipeline_id.

---

## 6. Conversations

### 6.1 `ai_conversation`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | RLS |
| `assistant_id` | UUID | |
| `user_id` | UUID | |
| `title` | VARCHAR(200) NULL | |
| `status` | VARCHAR(20) | |
| `token_input_total` | BIGINT | |
| `token_output_total` | BIGINT | |
| `correlation_id` | VARCHAR(64) NULL | |

### 6.2 `ai_message`

| Column | Type | Notes |
|---|---|---|
| `conversation_id` | UUID | |
| `role` | VARCHAR(20) | |
| `ordinal` | INT | |
| `content_redacted` | TEXT NULL | Stored form |
| `content_hash` | VARCHAR(64) NULL | |
| `model_deployment_id` | UUID NULL | |
| `latency_ms` | INT NULL | |
| `token_input` / `token_output` | INT NULL | |

Raw unredacted may be omitted or stored in encrypted media per policy — never world-readable logs.

### 6.3 `ai_citation`

| Column | Type | Notes |
|---|---|---|
| `message_id` | UUID | |
| `chunk_id` | UUID NULL | |
| `media_id` | UUID NULL | |
| `title` | VARCHAR(200) NULL | |
| `snippet` | TEXT NULL | |
| `score` | NUMERIC NULL | |

---

## 7. Corpora & embeddings

### 7.1 `ai_corpus`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `corpus_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `ingest_policy_id` | UUID | |
| `vector_index_ref` | VARCHAR(120) NULL | |
| `lifecycle` | VARCHAR(20) | |

### 7.2 `ai_chunk`

| Column | Type | Notes |
|---|---|---|
| `document_id` | UUID | |
| `ordinal` | INT | |
| `content_hash` | VARCHAR(64) | |
| `token_count` | INT NULL | |
| `vector_id` | VARCHAR(120) NULL | External vector id |
| `metadata` | JSONB | entity refs, path |

### 7.3 `ai_embed_job`

| Column | Type | Notes |
|---|---|---|
| `corpus_id` | UUID | |
| `status` | VARCHAR(20) | |
| `job_id` | UUID NULL | p14 |
| `docs_total` / `docs_done` | INT | |
| `error_message` | TEXT NULL | |

---

## 8. Tools

### 8.1 `ai_tool`

| Column | Type | Notes |
|---|---|---|
| `tool_key` | VARCHAR(100) UNIQUE | `trips.search` |
| `name` | VARCHAR(150) | |
| `risk_class` | VARCHAR(20) | |
| `confirm_policy` | VARCHAR(20) | |
| `handler_ref` | VARCHAR(200) | Domain gateway |

### 8.2 `ai_tool_version`

JSON Schema for parameters; `permission_codes[]`.

### 8.3 `ai_tool_invocation`

| Column | Type | Notes |
|---|---|---|
| `message_id` | UUID | |
| `tool_id` | UUID | |
| `status` | VARCHAR(20) | `PROPOSED`, `CONFIRMED`, `RUNNING`, `SUCCEEDED`, `DENIED`, `FAILED` |
| `args_json` | JSONB | Validated |
| `result_summary` | TEXT NULL | Redacted |

---

## 9. Guardrails & usage

### 9.1 `ai_safety_profile`

Bundles guardrail rules + redaction policy + max tokens + allow vision.

### 9.2 `ai_usage_event`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID NULL | |
| `assistant_id` | UUID NULL | |
| `deployment_id` | UUID NULL | |
| `modality` | VARCHAR(20) | |
| `token_input` / `token_output` | INT | |
| `embed_units` | INT NULL | |
| `cost_estimate` | NUMERIC NULL | |
| `occurred_at` | TIMESTAMPTZ | |

### 9.3 `ai_quota_policy`

| Column | Type | Notes |
|---|---|---|
| `scope` | VARCHAR(20) | `TENANT`, `USER`, `ASSISTANT` |
| `period` | VARCHAR(20) | `DAY`, `MONTH` |
| `max_token_in` / `max_token_out` | BIGINT NULL | |
| `max_embed_units` | BIGINT NULL | |
| `entitlement_code` | VARCHAR(100) NULL | p26 |

---

## 10. Eval & packs

- Suites with expected grounded answers / tool calls  
- Feedback: rating, comment, message_id  
- Packs: `copilot.ops.v1`, `copilot.finance.ask.v1`, `rag.policy.docs.v1`  

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `ai_outbox` | Domain events |
| `ai_idempotency_key` | Invoke / embed |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| System models/assistants | Read auth; manage admin |
| Tenant assistants/corpora | FORCE `tenant_id` |
| Conversations/messages | FORCE tenant + owner (or share) |
| Usage | FORCE tenant; admin aggregate |

---

## 13. Seed minimum

1. Models stubs: `chat.default`, `embed.default`  
2. Safety profile `standard` + `regulated`  
3. Assistant `ops.help` (draft) with NONE grounding  
4. Tools: `navigation.help` (LOW/AUTO) sample  
5. Quota: tenant monthly token cap placeholder  
6. Permissions `ai.*`  
7. Package `core.samples.v1`  

---

## 14. ER overview

```text
provider ── deployments ── models
assistant ── versions ── prompts / tools / corpora / safety
conversation ── messages ── citations / tool_invocations
corpus ── documents ── chunks ── embed_jobs → vector_index_ref
rag_pipeline ── runs
usage_events / quotas
eval_suites / feedback / packages
```

---

## 15. Implementation notes

1. Gateway resolves deployment via routing policy then fallback.  
2. Chunk vectors are external; deleting chunk must delete vector via gateway.  
3. HITL: tool stays `PROPOSED` until confirm API.  
4. Split models: `gateway`, `assistant`, `conversation`, `corpus`, `rag`, `tool`, `safety`, `usage`, `eval`, `governance`, `plumbing`.
