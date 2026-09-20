# JeslotERP AI Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — assistant/conversation HTTP persists on Postgres; empty list is `[]`. OpenAI live class is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p27_ai`  
**PostgreSQL schema:** `ai`  
**Depends on:** `p08_file_media`, `p18_search`, `p22_api`  
**Integrates with:** `p01_identity`, `p02_organization`, `p03_configuration`, `p05_metadata`, `p06_localization`, `p11_rules` (deterministic ≠ generative), `p12_feature`, `p13_event_bus`, `p14_messaging`, `p15_notification`, `p16_cache`, `p17_scheduler`, `p19_audit`, `p20_logging`, `p21_monitoring`, `p23_integration` (provider adapters), `p24_reporting`, `p26_licensing`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`AI_SCHEMA.md`](AI_SCHEMA.md) · [`AI_API.md`](AI_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise AI plane: model gateway, assistants, prompts, conversations, embeddings/RAG, tools, guardrails, evaluations, usage/cost, entitlements. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-022: assistant/conversation HTTP Postgres-first; empty `[]`; `require_ai_access`. `OpenAiModelGateway` fails `PROVIDER_PENDING` (no invented completions). |

---

## 1. Purpose (enterprise)

`p27_ai` is JeslotERP’s **AI assistants, embeddings, and model-gateway control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Joule / generative AI hub patterns** — governed models, grounding, enterprise prompts  
- **Microsoft Dynamics Copilot / Azure OpenAI enterprise patterns** — assistants, plugins, data grounding, quotas  
- **Salesforce Einstein / Agentforce-class boundaries** — grounded actions, trust layer, usage  
- **Banking / regulated ERP copilots** — audit, redaction, human-in-the-loop, no silent mutations  

It is **not** “call OpenAI from a React page.” It is the system that makes ERP AI correct for:

1. **Model catalog & gateway** — providers, deployments, routing, fallback  
2. **Assistants / agents** — system prompts, tools, grounding policies  
3. **Prompt library** — versioned templates, locales  
4. **Conversations** — threads, messages, streaming sessions  
5. **Embeddings & corpora** — chunking, vector indexes (via p18), media sources (p08)  
6. **RAG pipelines** — retrieve → ground → generate with citations  
7. **Tool calling** — allowlisted ERP actions (never arbitrary)  
8. **Guardrails** — PII redaction, topic allow/deny, jailbreak filters  
9. **Usage & cost** — tokens, quotas, tenant metering (align p26)  
10. **Eval & feedback** — golden sets, thumbs, regression gates  

### Owns

| Domain | Examples |
|---|---|
| Models / deployments | GPT/local/vendor endpoints |
| Gateway | Route, fallback, timeout |
| Assistants | Copilot defs |
| Prompts | Templates, versions |
| Conversations | Threads, messages |
| Embeddings / corpora | Docs, chunks |
| RAG | Pipelines, citations |
| Tools | Function registry |
| Guardrails | Policies |
| Usage / quotas | Token meters |
| Evals / feedback | Quality |
| Packs | Freight/finance copilots |

### Does **not** own

| Concern | Owner |
|---|---|
| Deterministic business decisions | `p11_rules` |
| Operational search index engine | `p18_search` (AI writes embeddings / queries via gateway) |
| File bytes | `p08_file_media` |
| Public API product keys/plans | `p22_api` (registers AI HTTP surface) |
| Commercial entitlements | `p26_licensing` |
| External HTTP plumbing | `p23_integration` (optional provider adapter) |
| Silent domain writes | Domain commands via tools only |

### Critical splits

| | **AI (p27)** | **Rules (p11)** | **Search (p18)** |
|---|---|---|---|
| Purpose | Generative assist / RAG | Deterministic decisions | Find / facets / vectors store |
| Output | Text, structured suggest | Allow/deny/score | Hits |
| Authority | Advisory unless tool committed | Binding when invoked | Retrieval only |
| Mutations | Via allowlisted tools | Via domain after decision | None |

**Rule:** Pricing/GST hard rules stay in **p11**/domain. Copilot may *suggest* and call tools that run real commands with AuthZ.

**Trust rule:** Models are untrusted; **gateway + guardrails + tool allowlists + RLS** are the trust boundary.

---

## 2. Architectural position

```text
Client / ERP shell
        │  (p22 catalog + p01 auth)
        ▼
  AI gateway (p27)
   │ route model · quota · guardrail
   ├─► assistant + prompt version
   ├─► RAG: corpus → embed → p18 retrieve → cite
   ├─► tools → domain commands (AuthZ)
   └─► provider via p23 / native adapter
        │
   usage meter → p26/p21 · audit meta → p19
```

**Hard rules**

1. No browser→vendor API keys; all inference through **p27 gateway**.  
2. Tool calls require **permission + confirmation policy** (auto vs HITL).  
3. RAG only from **authorized corpora** (tenant RLS); citations required for grounded answers when policy says.  
4. Prompts/responses may be retained under policy; **redact PII** for logs.  
5. Embeddings store vectors in search/vector backend; PG holds meta.  
6. Quotas fail-closed when exceeded.  
7. No cross-schema FKs.  
8. Feature/license gates before expensive models.

---

## 3. Advanced design principles

1. **Gateway-first** — single ingress for chat/complete/embed.  
2. **Deployment ≠ model card** — logical model + physical endpoint.  
3. **Assistant as product** — versioned, pack-shipped.  
4. **Prompt as code** — review, checksum, locale.  
5. **Grounding modes** — NONE | CORPUS | ENTITY | MIXED.  
6. **Citation discipline** — answer + sources[].  
7. **Tool allowlists** — per assistant, per tenant.  
8. **HITL** — high-risk tools need confirm.  
9. **Streaming** — SSE/websocket session ids.  
10. **Idempotent embeds** — content hash skip.  
11. **Eval gates** — block promote on regression.  
12. **Cost attribution** — tenant/user/assistant.  
13. **Model fallback chain** — primary → secondary.  
14. **Safety profiles** — regulated vs internal.  
15. **CQRS** — admin defs vs runtime invoke.  
16. **Outbox** — `ai.conversation.completed`, `ai.tool.invoked`.  
17. **Cache** — embedding + frequent RAG chunks (p16).  
18. **Observability** — latency/tokens/errors → p21.

---

## 4. Core concepts

### 4.1 Model & deployment

Catalog entry (capabilities, context window) + deployment (endpoint, secret_ref, region).

### 4.2 Assistant

Named copilot: prompt, tools, grounding corpus, safety profile, default model.

### 4.3 Conversation / thread

User session: messages (role system/user/assistant/tool), token totals, status.

### 4.4 Corpus & chunk

Governed document set from media/entities; chunking strategy; embed job.

### 4.5 RAG pipeline

Retrieve top-k → rerank → pack context → generate → cite.

### 4.6 Tool

Declared function: name, JSON schema, permission, risk class, handler ref.

### 4.7 Guardrail

Pre/post filters: PII, injection, topic, max output, blocked patterns.

### 4.8 Usage ledger

Token in/out, embeddings units, cost estimate, quota counters.

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| Auth / AuthZ | p01 on invoke + tool permissions |
| API exposure | Register ops in p22 |
| Files for RAG | p08 media → ingest |
| Retrieval | p18 vector/keyword query |
| Provider HTTP | p23 connection or native adapter |
| Quotas / SKUs | p26 entitlement hooks |
| Features | p12 model/assistant flags |
| Async embed/ingest | p14 jobs |
| Schedule reindex | p17 |
| User notify (job done) | p15 |
| Audit tool commits | p19 |
| Settings | p03 temperature defaults etc. |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `ai.catalog.read` | Models/assistants browse |
| `ai.assistant.manage` | Assistants/prompts |
| `ai.corpus.manage` | Corpora/ingest |
| `ai.invoke` | Chat/complete |
| `ai.embed` | Embedding APIs |
| `ai.tool.manage` | Tool registry |
| `ai.guardrail.manage` | Safety profiles |
| `ai.eval.manage` | Evals |
| `ai.usage.read` | Meters |
| `ai.admin` | Packs, deployments, quotas |
| `ai.*` | Wildcard |

### RLS

FORCE RLS on conversations, corpora, usage, tenant assistants.  
System assistants readable when published.

### Data handling

- Strip secrets from prompts via detectors.  
- Tool args validated against schema.  
- Conversation export respects retention + legal hold hooks.  
- Providers receive minimized context only.

---

## 7. Module layout

```text
platforms/p27_ai/
  application/
    services/
      model_gateway.py
      assistant_runtime.py
      prompt_renderer.py
      conversation_service.py
      embedding_service.py
      rag_pipeline.py
      tool_dispatcher.py
      guardrail_engine.py
      usage_meter.py
      eval_runner.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… providers/ persistence/ search_gateway/ media_gateway/
  tests/unit/gateway/ rag/ tools/ guardrail/ prompt/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `ai.assistant.published` | Catalog |
| `ai.conversation.started` / `completed` | Runtime |
| `ai.tool.invoked` / `confirmed` / `denied` | Tools |
| `ai.corpus.indexed` | Embeddings |
| `ai.guardrail.blocked` | Safety |
| `ai.quota.exceeded` | Usage |
| `ai.eval.completed` | Quality |
| `ai.deployment.unhealthy` | Gateway |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, permissions, model catalog |
| P2 | Gateway complete/chat + usage |
| P3 | Assistants + prompts |
| P4 | Conversations + streaming |
| P5 | Embeddings + corpus + p18 |
| P6 | RAG + citations |
| P7 | Tools + HITL |
| P8 | Guardrails + evals + packs |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] No client-held provider API keys  
- [ ] Quota exceeded → 429 fail-closed  
- [ ] Tool invoke checks permission + risk policy  
- [ ] Grounded mode returns citations or fails policy  
- [ ] Tenant cannot read other tenants’ threads/corpora  
- [ ] PII redaction applied to persisted logs per profile  
- [ ] Embed jobs idempotent on content hash  
- [ ] No cross-schema FKs  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Call OpenAI from frontend | p27 gateway |
| Let model invent ERP writes | Allowlisted tools + AuthZ |
| Use p11 as LLM prompt store | p11 deterministic; p27 prompts |
| Dump full document PII into vendor logs | Redact + minimize |
| Unlimited tokens per tenant | Quotas + p26 |
| RAG over all tenants’ media | Corpus RLS |

---

## 12. Related documents

- Schema: [`AI_SCHEMA.md`](AI_SCHEMA.md)  
- API: [`AI_API.md`](AI_API.md)  
- File media: [`../08_file_media/FILE_MEDIA_GUIDE.md`](../08_file_media/FILE_MEDIA_GUIDE.md)  
- Search: [`../18_search/SEARCH_GUIDE.md`](../18_search/SEARCH_GUIDE.md)  
- API platform: [`../22_api/API_GUIDE.md`](../22_api/API_GUIDE.md)  
- Rules: [`../11_rules/RULES_GUIDE.md`](../11_rules/RULES_GUIDE.md)  
- Licensing: [`../26_licensing/LICENSING_GUIDE.md`](../26_licensing/LICENSING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
