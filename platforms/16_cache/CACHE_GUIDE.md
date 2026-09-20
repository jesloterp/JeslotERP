# JeslotERP Cache Platform — Developer Integration Guide

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — namespace/TTL catalog Postgres-first; Redis L2 is `PROVIDER_PENDING` until attached. Cache is never the system of record. Not Production.  
**Package:** `platforms.p16_cache`  
**PostgreSQL schema:** `cache`  
**Depends on:** `p01_identity`, `p03_configuration`  
**Integrates with:** all read-heavy platforms (`p05`, `p06`, `p11`, `p12`, `p13`, …), `p14_messaging` (warmup/invalidate jobs), `p17_scheduler`, `p21_monitoring`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`CACHE_SCHEMA.md`](CACHE_SCHEMA.md) · [`CACHE_API.md`](CACHE_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise cache control plane: namespaces, key standards, TTL tiers, multi-layer (L1/L2), tags, invalidation bus, stampede locks, warmup, quotas, encryption refs, metrics, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-001: Postgres control-plane persist/fetch. |
| 1.2 | 2026-09-12 | TASK-SOR-026: buffer discipline — empty catalog is `[]`; Redis test-connection `PROVIDER_PENDING`; `buffer_class=NONE` refuses set; honest L2 layer. |

---

## 1. Purpose (enterprise)

`p16_cache` is JeslotERP’s **caching control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **Salesforce Platform Cache** — org/session partitions, TTL, capacity discipline  
- **Microsoft Dynamics / Azure Cache for Redis patterns** — distributed cache + invalidation  
- **SAP application buffers / OData cache discipline** — keyed buffering with controlled refresh  
- **Modern CDN + Redis meshes** — tags, purge, stampede protection  

It is **not** “developers call Redis however they want.” It is the system that makes ERP caching correct for:

1. **Governed namespaces** (`feature`, `i18n`, `metadata`, `rules`, `session_aux`)  
2. **Key standards** — templates, tenant scoping, version suffixes  
3. **TTL & tier policies** — volatile vs stable catalogs  
4. **Tag-based invalidation** — purge by `tenant:{id}`, `flag:{key}`, `doc:{id}`  
5. **Multi-layer** — L1 in-process + L2 Redis/Memcached (+ optional L3)  
6. **Stampede protection** — single-flight / soft TTL / lock  
7. **Warmup & prefetch** jobs after publish events  
8. **Tenant isolation & quotas** — noisy-neighbor control  
9. **Sensitive entry encryption** refs (secrets never as plaintext keys)  
10. **Hit/miss/latency metrics** and invalidation audit  

### Owns

| Domain | Examples |
|---|---|
| Namespaces / partitions | org, session, request, module |
| Key schemas | templates, validators |
| Policies | TTL, size, eviction hints |
| Backends | Redis clusters, local memory |
| Tags & invalidation | purge APIs, bus |
| Single-flight locks | stampede control |
| Warmup plans | post-deploy / post-publish |
| Quotas | per tenant/namespace |
| Observability | stats, samples |
| Governance | packs, approvals for prod purge |

### Does **not** own

| Concern | Owner |
|---|---|
| Business source of truth | Domain DBs |
| HTTP ETag generation per API | Each platform (may use p16) |
| CDN edge config for media | `p08_file_media` / infra |
| Session auth tokens | `p01_identity` (may use cache as store backend via policy) |
| Broker job execution | `p14_messaging` |

### Critical split: Cache vs Configuration vs Source data

| | **Cache (p16)** | **Configuration (p03)** | **Domain DB** |
|---|---|---|---|
| Durability | Ephemeral | Durable settings | System of record |
| Example | compiled rules blob | `cache.redis.url` secret | rules tables |
| Loss impact | Recompute | Misconfig | Business loss |

**Rule:** If losing the entry is unacceptable, it does **not** belong only in cache.

**Buffer discipline (JeslotERP buffer class):** `GENERIC` (default) may sit in L1/L2. `NONE` refuses `cache_set` (legal/audit SoR must not be buffered). L2 hits report `L2_MEMORY` unless a live Redis backend is actually attached — never invent `L2_REDIS`. Redis ping/test-connection is `PROVIDER_PENDING` until `CACHE_L2_PROVIDER` is set and ping succeeds. Pytest stays in-process MEMORY.

---

## 2. Architectural position

```text
Platform service
    │
    ├─ get_or_load(namespace, key, loader)
    │       ├─ L1 hit?
    │       ├─ L2 hit? → populate L1
    │       └─ miss → single-flight loader → set L2/L1 + tags
    │
    └─ invalidate(tags|keys) → bus → all nodes drop L1/L2
```

**Hard rules**

1. Keys always include **namespace** + **tenant** (when tenant-scoped).  
2. No unbounded `KEYS *` in production — use tag index / catalog.  
3. Invalidation is **explicit** on publish/activate events.  
4. Secrets/tokens: encrypt-at-rest policy or don’t cache.  
5. RLS/authz still applied on loader — cache is not an authz bypass.  
6. Cross-schema: UUID in values only; cache meta in `cache` schema.

---

## 3. Advanced design principles

1. **Namespace-first** — policy hangs off namespace, not ad-hoc keys.  
2. **Key templates** — `meta:entity:{tenant}:{entity_key}:v{ver}`  
3. **Versioned keys** — bump on schema change instead of mass scan.  
4. **Tags mandatory** for tenant-scoped entries.  
5. **Soft TTL + hard TTL** — serve stale while revalidate (optional).  
6. **Single-flight** — one loader per key across workers.  
7. **Write policies** — cache-aside (default), read-through, write-through (rare).  
8. **Null caching** — short TTL for misses to prevent hammering.  
9. **Compression** — large JSON blobs.  
10. **Size limits** — reject oversized values.  
11. **Propagation** — invalidate via pub/sub + outbox for reliability.  
12. **Warmup plans** — critical namespaces after deploy.  
13. **Environment separation** — never share prod Redis with staging.  
14. **Idempotent invalidate**.  
15. **Admin purge guarded** in production.  
16. **CQRS HTTP** — thin control APIs; hot path is library SDK.  
17. **Metrics labels** — namespace, tenant (sampled), result.  
18. **Packs** — seed namespace policies for platforms.

---

## 4. Core concepts

### 4.1 Namespace

```text
namespace_key = "rules.compiled"
partition = ORG | TENANT | SESSION | REQUEST | USER
default_ttl_sec, max_ttl_sec, max_value_bytes,
consistency = EVENTUAL | STRONG_INVALIDATION
sensitivity = NORMAL | SENSITIVE
```

### 4.2 Key

Built from template + params; hashed form stored optionally for tag maps.

### 4.3 Entry metadata (control plane)

p16 may store **policy & invalidation indexes** in Postgres; **values** live in Redis/L1.  
Optional `cache_entry_meta` for audited critical keys (not full payload).

### 4.4 Tags

```text
tenant:{uuid}
company:{uuid}
flag:{key}
rule:{key}
i18n:locale:{code}
user:{uuid}
```

Invalidate tag → delete all member keys (index maintained on set).

### 4.5 Layers

| Layer | Scope | Latency | Coherence |
|---|---|---|---|
| L1 | Process | µs–ms | Invalidate bus |
| L2 | Cluster Redis | ms | Shared |
| L3 | Optional regional | higher | Explicit |

### 4.6 Single-flight

```text
miss → try acquire lock(key, ttl)
  won → load → set → release → return
  lost → wait/poll or return soft-stale
```

### 4.7 Invalidation bus

Local outbox + Redis pub/sub / p13 event `cache.invalidate.v1` for multi-instance L1 drop.

---

## 5. Integration patterns

| Platform | Cache use |
|---|---|
| p05 metadata | Effective UI packs |
| p06 i18n | Locale bundles |
| p11 rules | Compiled artifacts |
| p12 feature | Flag env configs / bootstrap fragments |
| p13 | Subscription topology |
| p01 | Permission resolution (careful TTL) |

On `*.activated` / `*.published` → invalidate related tags.

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `cache.catalog.read` | Namespaces/policies |
| `cache.catalog.manage` | Manage namespaces |
| `cache.invalidate` | Invalidate keys/tags |
| `cache.purge` | Broad purge (admin) |
| `cache.warmup` | Trigger warmup |
| `cache.stats.read` | Metrics |
| `cache.backend.manage` | Backend registry |
| `cache.pack.install` | Packs |
| `cache.audit.read` | Audit |
| `cache.*` | Wildcard |

### RLS

Tenant-scoped invalidation requests & stats samples FORCE `tenant_id`.  
System namespaces readable with auth.

**Important:** Cache get APIs for arbitrary keys are **not** exposed to end users — only control-plane + internal SDK.

---

## 7. Module layout

```text
platforms/p16_cache/
  application/
    services/
      key_builder.py
      policy_resolver.py
      cache_aside.py
      single_flight.py
      invalidator.py
      tag_index.py
      warmup.py
      metrics.py
    sdk/  # in-process client used by other platforms
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/…
    backends/ memory.py redis.py
    messaging/ invalidate_bus.py
  tests/unit/keys/ invalidate/ single_flight/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `cache.namespace.changed` | Policy |
| `cache.invalidate.requested` / `completed` | Purge |
| `cache.warmup.started` / `completed` | Warmup |
| `cache.backend.unhealthy` | Ops |
| `cache.quota.breached` | Tenant |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, namespaces, permissions |
| P2 | Redis adapter + cache-aside SDK |
| P3 | Tags + invalidate API |
| P4 | L1 + invalidation bus |
| P5 | Single-flight + soft TTL |
| P6 | Warmup jobs + metrics |
| P7 | Quotas + sensitive encryption |
| P8 | Packs for platform namespaces |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [x] Key template validation rejects unscoped tenant keys  
- [x] Tag invalidate removes indexed keys  
- [x] Single-flight prevents thundering herd in test  
- [x] L1 drops on invalidate bus message  
- [x] Production purge requires `cache.purge` + confirm  
- [x] Metrics expose hit ratio per namespace  
- [x] No secret values in Postgres meta tables  
- [x] No cross-schema FKs  
- [x] HTTP catalog persist on `AsyncSession` (empty list is `[]`); RLS on `require_cache_access`  
- [x] Redis L2 is a fail-closed port (`PROVIDER_PENDING`); pytest stays MEMORY  
- [x] `buffer_class=NONE` refuses cache_set (cache is never SoR)  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Cache without tenant in key | Template enforces tenant |
| `FLUSHALL` in prod casually | Tag/namespace purge |
| Cache authz decisions forever | Short TTL + invalidate on role change |
| Store PII plaintext “for speed” | Sensitivity policy / don’t cache |
| Skip invalidate on publish | Hook activate events |
| Let each module invent key strings | Registered templates |

---

## 12. Related documents

- Schema: [`CACHE_SCHEMA.md`](CACHE_SCHEMA.md)  
- API: [`CACHE_API.md`](CACHE_API.md)  
- Configuration: [`../03_configuration/CONFIGURATION_GUIDE.md`](../03_configuration/CONFIGURATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
