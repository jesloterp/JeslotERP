# JeslotERP API Platform — Developer Integration Guide

**Version:** 1.1  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — hashed API keys persist on Postgres; empty list is `[]`. Redis rate-limit attaches when reachable; pytest stays MEMORY. Bulk + Composite on p22. Not Production.  
**Package:** `platforms.p22_api`  
**PostgreSQL schema:** `api`  
**Depends on:** `p01_identity`, `p12_feature`  
**Integrates with:** all HTTP platforms, `p03_configuration`, `p13_event_bus`, `p14_messaging`, `p16_cache`, `p19_audit`, `p20_logging`, `p21_monitoring`, `p23_integration` (consumes external; p22 governs inbound/public product APIs)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`API_SCHEMA.md`](API_SCHEMA.md) · [`API_ENDPOINTS.md`](API_ENDPOINTS.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise API control plane: OpenAPI catalog, products/plans, keys/OAuth bindings, versioning/deprecation, rate limits/quotas, policies, idempotency standards, IP allowlists, portal meta, analytics. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-020: key HTTP Postgres-first (`key_hash` only); empty list is `[]`; `require_api_access` sets RLS GUCs. Redis rate-limit port already live-when-reachable. Bulk deferred. |
| **1.1** | **2026-09-12** | HYG-017: Bulk jobs + Composite subrequests on p22. OData/GraphQL not demanded. |

---

## 1. Purpose (enterprise)

`p22_api` is JeslotERP’s **API product & gateway control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP API Management / SAP Gateway** — API products, policies, consumers  
- **Microsoft Dynamics / Azure API Management** — products, subscriptions, rate limits, revisions  
- **Salesforce API / Connected Apps governance** — versioning, limits, client identity  
- **Kong / Apigee / AWS API Gateway class systems** — catalog, keys, plans, analytics  

It is **not** “FastAPI routers exist.” It is the system that makes ERP API exposure correct for:

1. **Canonical API catalog** — services, routes, OpenAPI specs  
2. **Versioning & deprecation** — `v1`/`v2`, sunset headers, migration notes  
3. **API products & plans** — what partners/tenants can call  
4. **Credentials** — API keys, OAuth client bindings (secrets hashed / via identity)  
5. **Rate limits & quotas** — per key/plan/tenant/route  
6. **Policies** — auth required, idempotency, IP allowlist, CORS, payload size  
7. **Feature-gated routes** via p12  
8. **Consumer analytics** — usage, top routes, error rates  
9. **Developer portal metadata** — docs links, try-it scopes  
10. **Standards** — `StandardResponse`, idempotency, correlation headers  

### Owns

| Domain | Examples |
|---|---|
| Catalog | services, routes, operations |
| Specs | OpenAPI versions |
| Products / plans | partner tiers |
| Credentials | API keys, client bindings |
| Rate limits / quotas | policies & counters meta |
| Gateway policies | authz, CORS, IP, size |
| Versioning | revisions, deprecations |
| Usage analytics | aggregates |
| Portal | public doc metadata |
| Governance | packs, approvals |
| Bulk / Composite | partner batch + referenced subrequests (not OData/GraphQL) |

### Does **not** own

| Concern | Owner |
|---|---|
| User login / JWT issuance | `p01_identity` |
| External outbound connectors | `p23_integration` |
| Business handlers | Domain platforms |
| WAF / TLS termination | Infra |
| Feature flag evaluation | `p12_feature` (consulted) |

### Critical split: API platform vs Identity vs Integration

| | **API (p22)** | **Identity (p01)** | **Integration (p23)** |
|---|---|---|---|
| Focus | Productize & govern HTTP APIs | Who is the principal | Call external systems |
| Credential | API key / subscription | User/session/OAuth tokens | Connector secrets |
| Direction | Inbound (mostly) | AuthN/Z | Outbound (+ inbound webhooks) |

**Rule:** Route handlers live in platforms; **p22 decides exposure, plan, limit, version**.

---

## 2. Architectural position

```text
Client / Partner / Mobile
        │
        ▼
  Edge / Gateway (enforces p22 policies)
        │  authn (p01) · key · plan · rate limit · feature
        ▼
  Platform routers (/api/v1/…)
        │
        ▼
  usage meters → p22 analytics (+ p21 metrics)
```

**Hard rules**

1. Production public routes must be **registered** in catalog (CI gate).  
2. API keys stored **hashed**; plaintext shown once.  
3. Rate limit decisions must be **fast** (Redis/cache); PG holds policy.  
4. Deprecation requires `Sunset` / `Deprecation` headers policy.  
5. No cross-schema FKs.  
6. RLS on tenant subscriptions & keys.

---

## 3. Advanced design principles

1. **Operation-first catalog** — `method + path + operation_id`.  
2. **OpenAPI as contract** — publish checksummed specs.  
3. **Products bundle operations**.  
4. **Plans** attach limits & entitlements.  
5. **Subscriptions** bind tenant/consumer → plan.  
6. **Keys** bind to subscription; optional user.  
7. **Multi-dimensional limits** — RPS, daily quota, concurrency, payload.  
8. **Idempotency policy** — required methods/routes.  
9. **Revision vs version** — revision for non-breaking; version for breaking.  
10. **Soft deploy** — catalog active before traffic switch.  
11. **IP allowlists** per subscription.  
12. **CORS policies** per product.  
13. **Changelog** per version.  
14. **Try-it** scopes gated by feature + plan.  
15. **Analytics** sampled request logs meta (not full bodies).  
16. **CQRS HTTP** for admin; gateway reads cached snapshot.  
17. **Packs** — seed core public/internal product definitions.  
18. **Audit** key create/revoke via p19.

---

## 4. Core concepts

### 4.1 Service & operation

```text
service_key = "identity" | "document" | …
operation_id = "createUser"
method = POST
path_template = /api/v1/users
```

### 4.2 Product / plan / subscription

```text
product "Partner ERP API"
  plan FREE | STANDARD | ENTERPRISE
    limits: 10 rps / 100k day
subscription (tenant X) → ENTERPRISE
  api_keys[]
```

### 4.3 Version lifecycle

```text
DRAFT → PUBLISHED → ACTIVE → DEPRECATED → RETIRED
```

### 4.4 Gateway policy bundle

Auth mode (JWT/API_KEY/MUTUAL), required scopes/permissions, idempotency, max body, feature flags, IP policy.

### 4.5 Rate limit result

```text
Allow / Deny
Remaining, Reset, Retry-After
```

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| JWT validation | p01 JWKS / introspect |
| Permission check | p01 authz after authn |
| Feature route | p12 evaluate |
| Usage metrics | p21 counters |
| Key revoke audit | p19 |
| Partner webhooks outbound | p23 (not p22) |

Gateway snapshot published to cache (`api.gateway.snapshot`) via p16 tags.

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `api.catalog.read` | Read catalog/specs |
| `api.catalog.manage` | Manage routes/specs |
| `api.product.manage` | Products/plans |
| `api.key.manage` | Issue/revoke keys |
| `api.subscription.manage` | Subscriptions |
| `api.limit.manage` | Rate policies |
| `api.portal.manage` | Portal meta |
| `api.analytics.read` | Usage |
| `api.admin` | Packs/snapshots |
| `api.*` | Wildcard |

### RLS

FORCE RLS on tenant subscriptions, keys, usage.  
Catalog system-readable.

---

## 7. Module layout

```text
platforms/p22_api/
  application/
    services/
      catalog.py
      openapi_registry.py
      product_plan.py
      key_service.py
      rate_limiter.py
      policy_compiler.py
      gateway_snapshot.py
      analytics.py
      deprecation.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… cache/ gateway_adapter/
  tests/unit/limit/ key/ openapi/ policy/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `api.spec.published` / `version.deprecated` | Catalog |
| `api.key.issued` / `revoked` | Credentials |
| `api.subscription.changed` | Plans |
| `api.limit.violated` | Abuse (sampled) |
| `api.snapshot.published` | Gateway |
| `api.product.activated` | Products |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, catalog, permissions |
| P2 | OpenAPI register + version |
| P3 | Products/plans/subscriptions |
| P4 | API keys + hash verify |
| P5 | Rate limit engine + headers |
| P6 | Policy compiler + snapshot |
| P7 | Deprecation + analytics |
| P8 | Portal meta + packs |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] Unregistered prod route fails CI catalog check  
- [ ] Key plaintext shown once; verify uses hash  
- [ ] Rate limit deny returns 429 + Retry-After  
- [ ] Deprecated version emits Sunset header policy  
- [ ] Snapshot invalidate on plan/key change  
- [ ] Tenant cannot see other tenants’ keys  
- [ ] Feature-gated operation denied when flag off  
- [ ] No cross-schema FKs  
- [x] HYG-017: `POST /composite` + Bulk job close/abort; no `/odata` or GraphQL  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Ship routes without catalog | Register operation_id |
| Store API keys plaintext | Hash + one-time show |
| One global RPM for all partners | Plans/subscriptions |
| Breaking change in v1 silently | New version + deprecation |
| Put business logic in gateway | Policies only; handlers in platforms |
| Log full request bodies always | Sampled meta analytics |

---

## 12. Related documents

- Schema: [`API_SCHEMA.md`](API_SCHEMA.md)  
- Endpoints: [`API_ENDPOINTS.md`](API_ENDPOINTS.md)  
- Identity: [`../01_identity/IDENTITY_GUIDE.md`](../01_identity/IDENTITY_GUIDE.md)  
- Feature: [`../12_feature/FEATURE_GUIDE.md`](../12_feature/FEATURE_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
