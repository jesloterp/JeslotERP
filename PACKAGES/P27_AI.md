# AI (`p27_ai`)

**Package:** `p27_ai`  
**Schema:** `ai`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Model gateway, assistants, conversations, tools, embeddings / RAG, guardrails, evaluation, and usage.

## 2. Responsibilities

- Own the `ai` persistence schema and the `p27_ai` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Models, deployments, routing
- Assistants and prompts
- Chat / complete / stream and conversations
- Tools with human confirm / deny
- Embeddings, corpora, ingest jobs
- RAG pipelines and query
- Safety profiles and guardrails
- Usage, quotas, cost, eval suites
- Default development / stub provider
- External model provider path fails closed without a key

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Model deployment**
- **Assistant**
- **Conversation**
- **Tool invocation**
- **Corpus**
- **Embed job**
- **RAG pipeline**
- **Safety profile**
- **Guardrail event**
- **Usage rollup**

## 5. Major Capabilities

- Models, deployments, routing
- Assistants and prompts
- Chat / complete / stream and conversations
- Tools with human confirm / deny
- Embeddings, corpora, ingest jobs
- RAG pipelines and query
- Safety profiles and guardrails
- Usage, quotas, cost, eval suites
- Default development / stub provider
- External model provider path fails closed without a key

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Gateway
- Assistant
- Conversation
- Corpus / RAG
- Tool
- Safety
- Usage
- Eval
- Outbox

Public API resource groups:

- Models / assistants / chat / conversations / tools / corpora / RAG / safety / usage / eval
- Internal embed jobs

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p08_file_media`
- `p18_search`
- `p22_api`

### Additional verified runtime coupling

- Media gateway for corpus sources
- Search for vector / chunk index

### Recommended future dependency

- p26_licensing quota hooks
- human-in-the-loop tool confirm

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Operator assistants
- future document Q&A

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `ai.assistant.published`
- `ai.quota.exceeded`
- `ai.guardrail.blocked`
- `ai.conversation.started`
- `ai.corpus.indexed`
- `ai.eval.completed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Provider selection is environment-specific; isolated tests force the stub.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Tool confirm / deny
- Guardrails
- Quota events
- Tenant RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Model-gateway port
- Media port
- Vector index on p18_search

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `ai` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Production model vendor is environment-blocked
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

AI may assist posting and inquiry; it must not bypass rules, process, or audit.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
