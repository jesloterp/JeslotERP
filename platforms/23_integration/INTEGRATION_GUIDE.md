# JeslotERP Integration Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — outbound messages + delivery attempts persist on Postgres; empty attempt list is `[]`. Adapters stay ports (`stub` / `PROVIDER_PENDING`). Not Production.  
**Package:** `platforms.p23_integration`  
**PostgreSQL schema:** `integration`  
**Depends on:** `p13_event_bus`, `p14_messaging`, `p22_api`  
**Integrates with:** `p01_identity`, `p02_organization`, `p03_configuration`, `p04_business_partner`, `p05_metadata`, `p08_file_media`, `p12_feature`, `p15_notification` (user notify ≠ partner protocol), `p16_cache`, `p17_scheduler`, `p19_audit`, `p20_logging`, `p21_monitoring`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`INTEGRATION_SCHEMA.md`](INTEGRATION_SCHEMA.md) · [`INTEGRATION_API.md`](INTEGRATION_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise integration plane: connectors, connections, mappings, pipelines, outbound/inbound, webhooks, retries/circuit breakers, DLQ/replay, partner profiles, secrets refs, packs. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-021: outbound/attempt HTTP Postgres-first; empty list is `[]`; `require_integration_access` sets RLS GUCs. REST stub + pending connector ports; no invented SAP/Salesforce/GST clients. |

---

## 1. Purpose (enterprise)

`p23_integration` is JeslotERP’s **external system connectivity control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP PI/PO / CPI / Integration Suite** — adapters, mappings, monitoring, reprocessing  
- **Microsoft Dynamics Dual-write / Dataverse plugins / Logic Apps patterns** — connectors, transforms, retries  
- **Salesforce Connected Apps / Platform Events / MuleSoft-class iPaaS boundaries** — outbound + inbound webhooks  
- **Banking / GST / EDI gateways** — signed payloads, idempotent delivery, reconciliation  

It is **not** “call `requests.post` from a domain service.” It is the system that makes ERP integration correct for:

1. **Connector catalog** — REST, SOAP, SFTP, EDI, webhook, custom adapters  
2. **Connections** — tenant/partner endpoints, auth, TLS, secrets-by-ref  
3. **Mappings & transforms** — canonical ↔ external field maps  
4. **Pipelines / flows** — trigger → map → call → respond → reconcile  
5. **Outbound delivery** — enqueue (p14), retry, backoff, circuit breaker  
6. **Inbound webhooks** — verify signature, idempotency, normalize, emit (p13)  
7. **Dead-letter & replay** — operator reprocess with audit  
8. **Partner profiles** — per BP/tenant integration contracts  
9. **Observability** — latency, error classes, SLO hooks (p21)  
10. **Governance** — packs (e-invoice, payment, telematics), feature gates  

### Owns

| Domain | Examples |
|---|---|
| Connectors | Adapter types, capabilities |
| Connections | Endpoint instances |
| Credentials refs | Vault/config secret pointers |
| Mappings | Field/transform defs |
| Pipelines | Flow definitions |
| Outbound messages | Delivery attempts |
| Inbound webhooks | Endpoints, verifies |
| DLQ / replay | Failed payloads |
| Partner profiles | Contracts |
| Reconciliation | External ids, sync state |
| Packs | Industry connectors |

### Does **not** own

| Concern | Owner |
|---|---|
| Public product API catalog / API keys | `p22_api` |
| Domain event fabric | `p13_event_bus` |
| Generic job workers | `p14_messaging` (executes integration jobs) |
| End-user email/SMS/push copy | `p15_notification` |
| Business mutations | Domain platforms |
| Secret plaintext storage | Vault / `p03` secret store |

### Critical splits

| | **Integration (p23)** | **API (p22)** | **Messaging (p14)** | **Event Bus (p13)** |
|---|---|---|---|---|
| Direction | Mostly **outbound** + external **inbound webhooks** | Inbound product APIs | Internal work queue | Internal facts |
| Credential | Connection secrets / mTLS | API keys / plans | Job auth internal | N/A |
| Payload | External schema | Our OpenAPI | Job args | CloudEvents |
| Retry | Adapter-aware | Rate-limit client | Worker retry | Relay/outbox |

**Rule:** Domain services publish intent (`invoice.posted`) → p13 → integration pipeline maps & delivers to GST/payment. Partners never get raw DB access; they get governed connectors.

**vs Notification:** p15 tells *users*; p23 talks to *systems* (banks, GSTIRN, telematics, EDI VAN).

---

## 2. Architectural position

```text
Domain platform ──outbox──► p13 event
                              │
                              ▼
                    integration pipeline
                     map │ transform │ enrich
                              │
                    enqueue p14 job (integration.deliver)
                              │
                              ▼
                    connector adapter ──► External system
                              │
                         retry / CB / DLQ
                              │
                    reconcile + audit (p19) + metrics (p21)

External system ──HTTPS──► inbound webhook endpoint (p23)
                              │ verify · idempotent · normalize
                              ▼
                         p13 event / domain command gateway
```

**Hard rules**

1. No plaintext secrets in `integration` tables — **secret_ref** only.  
2. Outbound side effects go through **pipelines** (auditable), not ad-hoc HTTP in domains.  
3. Inbound webhooks are **fail-closed** on bad signature / unknown connection.  
4. Delivery is **at-least-once** with idempotency keys toward external + local.  
5. Heavy I/O via **p14**; p23 stores control state + attempt history.  
6. No cross-schema FKs (UUID + gateways).  
7. Circuit breaker opens protect partners from stampede.  
8. Payload bodies may be truncated/redacted in logs; full blobs in media if needed (p08).

---

## 3. Advanced design principles

1. **Connector ≠ connection** — type vs tenant instance.  
2. **Pipeline as product** — versioned, publishable flows.  
3. **Canonical messages** — map at edge; domains stay clean.  
4. **Idempotency everywhere** — inbound & outbound.  
5. **Retry taxonomy** — retryable vs poison vs auth-fatal.  
6. **Circuit breaker + bulkhead** per connection.  
7. **Rate limit toward external** — respect partner quotas.  
8. **Correlation** — `correlation_id` / external `request_id` preserved.  
9. **Reconciliation keys** — `external_object_id` store.  
10. **Webhook endpoint tokens** — rotate without downtime.  
11. **Sandbox vs live** connections.  
12. **Mapping DSL / JSONPath / JSONata-class** — sandboxed.  
13. **Binary/EDI** via media + parser adapters.  
14. **Feature packs** — enable GST/payment connectors via p12.  
15. **CQRS** admin vs runtime delivery APIs.  
16. **Outbox** for integration domain events.  
17. **Replay with same idempotency** or explicit new key (policy).  
18. **Partner-facing SLAs** tracked in attempt metrics.

---

## 4. Core concepts

### 4.1 Connector

Reusable adapter: `REST_JSON`, `SOAP`, `SFTP`, `WEBHOOK_IN`, `EDI_X12`, `CUSTOM`.

### 4.2 Connection

Bound endpoint: base URL, auth mode (API_KEY, OAUTH2, MTLS, BASIC), secret refs, env (`SANDBOX`|`LIVE`).

### 4.3 Mapping

Versioned transform: inbound/outbound direction, field map, constants, scripts (sandbox).

### 4.4 Pipeline

Trigger (event type / schedule / manual) → steps (map, call, branch, emit, notify-ops).

### 4.5 Delivery / Attempt

Outbound unit of work with status, HTTP code, latency, error class, next_retry_at.

### 4.6 Inbound reception

Webhook hit → verify → store reception → process → ack.

### 4.7 DLQ item

Poison or exhausted retries; replayable by operator with permission.

### 4.8 Partner profile

Links BP/tenant to allowed pipelines, connections, SLAs.

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| Trigger | p13 subscriptions / p17 schedule / manual API |
| Execute call | p14 job `integration.deliver` / `integration.inbound.process` |
| Secrets | p03 / vault `secret_ref` |
| Files | p08 for EDI/SFTP payloads |
| Partner master | p04 soft refs |
| Field semantics | p05 optional |
| Auth of admin APIs | p01 + p22 catalog registration |
| User alert on failure | p15 (ops) |
| Audit connection/mapping changes | p19 |
| Adapter latency/errors | p21 |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `integration.catalog.read` | Connectors/pipelines browse |
| `integration.connection.manage` | Connections |
| `integration.mapping.manage` | Mappings |
| `integration.pipeline.manage` | Pipelines |
| `integration.deliver` | Manual trigger |
| `integration.webhook.manage` | Inbound endpoints |
| `integration.dlq.manage` | Replay/purge DLQ |
| `integration.partner.manage` | Profiles |
| `integration.admin` | Packs, breakers reset |
| `integration.*` | Wildcard |

### RLS

FORCE RLS on tenant connections, deliveries, receptions, DLQ, partner profiles.  
System connector catalog readable.

### Secrets & payloads

- Secret values never returned by API.  
- Webhook raw bodies retained per retention policy; PII redaction rules.  
- mTLS certs referenced, not uploaded as PEM in PG when possible.

---

## 7. Module layout

```text
platforms/p23_integration/
  application/
    services/
      connector_registry.py
      connection_service.py
      mapping_engine.py
      pipeline_runtime.py
      outbound_deliverer.py
      inbound_webhook.py
      retry_policy.py
      circuit_breaker.py
      dlq_service.py
      reconcile_service.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… adapters/rest|soap|sftp|edi/ persistence/
  tests/unit/mapping/ retry/ webhook/ pipeline/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `integration.connection.activated` / `rotated` | Connection |
| `integration.pipeline.published` | Pipeline |
| `integration.delivery.succeeded` / `failed` | Outbound |
| `integration.inbound.received` / `processed` | Webhook |
| `integration.dlq.enqueued` / `replayed` | DLQ |
| `integration.circuit.opened` / `closed` | Breaker |
| `integration.mapping.published` | Mapping |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, permissions, connector catalog |
| P2 | Connections + secret refs |
| P3 | Mapping engine |
| P4 | Pipeline + outbound via p14 |
| P5 | Retry / CB / DLQ |
| P6 | Inbound webhooks |
| P7 | Partner profiles + reconcile |
| P8 | Packs (sample REST + webhook) |
| P9 | Registry → **SoR-Live** (attempts on Postgres; adapters remain ports) |

---

## 10. Definition of Done (enterprise)

- [x] Domain code has no direct external HTTP for governed flows (adapter ports only)  
- [x] Secrets never stored/logged plaintext  
- [x] Webhook reject invalid signature (401/403)  
- [x] Exhausted retries land in DLQ with replay API  
- [x] Circuit open stops new attempts; metrics emitted  
- [x] Idempotent inbound (duplicate key → 200 ack, no double apply)  
- [x] Tenant isolation on connections/deliveries (`require_integration_access` + FORCE RLS)  
- [x] No cross-schema FKs  
- [x] Deliver/attempts HTTP Postgres-first; empty catalog is `[]` (not memory leak) 

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| `httpx` inside domain command | Pipeline + connector |
| Store API keys in connection row | `secret_ref` |
| Infinite retries | Max attempts + DLQ |
| Treat p22 API keys as outbound secrets | Separate connection credentials |
| Use p15 for EDI/bank protocols | p23 adapters |
| Replay without audit | DLQ replay + p19 |

---

## 12. Related documents

- Schema: [`INTEGRATION_SCHEMA.md`](INTEGRATION_SCHEMA.md)  
- API: [`INTEGRATION_API.md`](INTEGRATION_API.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_GUIDE.md`](../13_event_bus/EVENT_BUS_GUIDE.md)  
- Messaging: [`../14_messaging/MESSAGING_GUIDE.md`](../14_messaging/MESSAGING_GUIDE.md)  
- API platform: [`../22_api/API_GUIDE.md`](../22_api/API_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
