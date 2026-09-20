# JeslotERP Metadata Platform — Developer Integration Guide

**Version:** 2.0 (Advanced)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** (HTTP dictionary / overlay / publish hit PostgreSQL when `session` is `AsyncSession`. Not Production.)  
**Package:** `platforms.p05_metadata`  
**PostgreSQL schema:** `metadata`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`  
**Optionally integrates:** `p06_localization`, `p12_feature`, `p18_search`, `p19_audit`, `p24_reporting`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`METADATA_SCHEMA.md`](METADATA_SCHEMA.md) · [`METADATA_API.md`](METADATA_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-09 | Initial dictionary + UI descriptor baseline. |
| **2.0** | **2026-09-09** | Advanced enterprise upgrade: layered effective resolution, FLS, impact/drift, semantic layer, packages, channel packs, expression AST, publish governance. |
| 2.1 | 2026-09-12 | TASK-SOR-016: HTTP catalog SoR (modules/entities/fields/overlays/publish) + RLS on access. |

---

## 1. Purpose (advanced)

`p05_metadata` is JeslotERP’s **enterprise metadata control plane** — comparable in ambition to well-known enterprise metadata systems (examples only: dictionaries and annotation models used in products from SAP, Microsoft, Salesforce, and Oracle). Those names are **reference patterns**, not affiliation or compatibility. See [TRADEMARKS.md](../../TRADEMARKS.md).

It is **not** a thin field catalog. It is the system that lets the ERP:

1. Describe every business object as a governed contract  
2. Drive **metadata-first UI** (forms, grids, inspectors, wizards, mobile)  
3. Enforce **declarative validation** before business APIs commit  
4. Allow **safe tenant extensibility** without forking core code  
5. Publish **immutable versions** with approval, diff, rollback  
6. Power **search, reporting, AI, and codegen** from one catalog  
7. Detect **schema drift** between live DDL and declared metadata  
8. Analyze **blast radius** before deprecating a field  

### Owns

| Domain | Examples |
|---|---|
| Data dictionary | modules, entities, fields, options, relations, indexes, constraints |
| Semantic layer | business domains, semantic types, measures, dimensions |
| UI descriptors | forms, sections, lists, filters, actions, channel variants, a11y |
| Behavior | validation rules, computed fields, defaults, visibility AST |
| Security descriptors | field classification, masking, FLS policies (hints → IAM) |
| Extensibility | custom fields, overlays, industry packs, tenant packages |
| Governance | changesets, approvals, publish versions, artifacts, rollback |
| Observability of catalog | dependency graph, impact analysis, drift reports, audit trail |
| Interop contracts | command/query/event descriptors, OpenAPI projection hints |

### Does **not** own

| Concern | Owner |
|---|---|
| Runtime setting **values** | `p03_configuration` |
| AuthN/AuthZ enforcement | `p01_identity` (Metadata only **declares** required permissions) |
| Org hierarchy | `p02_organization` |
| Translated strings | `p06_localization` (Metadata stores `label_key`) |
| Number series | `p07_number_series` |
| Physical DDL migrations | Alembic / owning platform |
| Business row data | Owning platforms / business modules |
| Arbitrary code execution | Forbidden — safe AST only |

### Configuration vs Metadata vs Domain

| Layer | Question | Example |
|---|---|---|
| **Configuration** | What is the value? | `default_page_rows = 50` |
| **Metadata** | What is the shape / UI / rules? | `bp.partner.gstin` is STRING, PII, regex… |
| **Domain API** | What is the record? | Partner row id=… gstin=27… |

---

## 2. Architectural position

```text
                    ┌─────────────────────────────────────────┐
                    │     EFFECTIVE METADATA RESOLVER          │
                    │  SYSTEM → PACK → TENANT → COMPANY →      │
                    │  ROLE → USER → CHANNEL → FEATURE_FLAG    │
                    └───────────────────┬─────────────────────┘
                                        │
     ┌──────────────┬───────────────────┼───────────────────┬──────────────┐
     ▼              ▼                   ▼                   ▼              ▼
 Frontend       Validate API      Search/Report          Codegen        AI tools
 (ui-pack)      (pre-commit)      (field catalog)     (DTOs/OpenAPI)  (embeddings)
```

**Hard rule:** Metadata is the **contract catalog**. Typed domain models + Alembic remain source of physical truth. Metadata may *detect* drift; it must not silently alter production DDL.

---

## 3. Advanced design principles

1. **Layered effective resolution** — never a single flat catalog at runtime.  
2. **Publish immutability** — published artifacts are content-addressed (checksum); rollback = publish previous.  
3. **Key stability** — `entity_key` / `field_key` immutable after first publish; use deprecate + replace.  
4. **Safe expression AST** — visibility/defaults/computed fields use allow-listed operators only (no `eval`, no SQL).  
5. **Field-level security descriptors** — classification + mask + required permission; IAM enforces.  
6. **Extensibility without fork** — tenants add fields/overlays; cannot delete system fields.  
7. **Package portability** — export/import YAML/JSON industry packs.  
8. **Impact before break** — dependency graph required for deprecate/publish gates.  
9. **Drift awareness** — compare declared native columns vs live `information_schema`.  
10. **Channel & density aware UI** — `WEB_DENSE`, `WEB_COMFORT`, `MOBILE`, `PRINT`, `ACCESSIBLE`.  
11. **Semantic + physical** — every field has data type *and* optional semantic type (`GSTIN`, `MONEY_INR`).  
12. **CQRS/event catalog** — document commands/queries/events per entity for platform coherence.  
13. **No cross-schema FKs** — UUID refs only.  
14. **RLS fail-closed** on tenant tables; FORCE RLS.  
15. **CQRS HTTP** — thin routers; application services for resolve/publish/validate/impact.

---

## 4. Effective resolution model (critical)

Runtime consumers call **effective** APIs. Resolver merges layers in order (later wins for overlays; extensions **add**):

| Priority | Layer | Source |
|---:|---|---|
| 10 | `SYSTEM` | Seeded product catalog |
| 20 | `INDUSTRY_PACK` | Installed pack (e.g. `india.gst`, `transport.ftl`) |
| 30 | `TENANT` | Tenant entities/fields/overlays |
| 40 | `COMPANY` | Company-scoped overlays |
| 50 | `ROLE` | Role-based form/list variants |
| 60 | `USER` | Personal column order / hidden columns (non-breaking) |
| 70 | `CHANNEL` | Web/mobile/print selection |
| 80 | `FEATURE_FLAG` | `p12_feature` gated fields/actions |

### Resolve inputs

```text
tenant_id, company_id, branch_id?,
user_id, role_codes[],
channel, locale?,
feature_flags[],
publish_version? (pin),
entity_key / form_key / list_key
```

### Resolve outputs

- Effective entity + fields (with origin layer tags)  
- Effective form/list/filter/actions  
- Applied mask policies (client must still obey; server redacts where required)  
- `etag` / `checksum` for HTTP caching  

---

## 5. Key advanced concepts

### 5.1 Semantic types

Physical `STRING` + semantic `GSTIN` enables shared validators, masks, and search analyzers without proliferating data types.

### 5.2 Computed / virtual fields

`storage_strategy = VIRTUAL` + `compute_ast` (safe AST). Evaluated in application layer or projected in read models — **never** arbitrary Python.

### 5.3 Conditional UI

`visibility_ast`, `required_ast`, `readonly_ast` on form fields — same AST engine.

### 5.4 Field-level security (FLS)

`metadata_field_security` declares:

- `classification` (PUBLIC / INTERNAL / CONFIDENTIAL / RESTRICTED / PII / SPI)  
- `mask_strategy` (NONE / LAST4 / HASH / REDACT / TOKENIZE)  
- `read_permission` / `write_permission`  

IAM remains enforcer; Metadata drives UI + response redaction helpers.

### 5.5 Dependency graph

Edges: field→entity, form_field→field, action→permission, validation→field, measure→field, event→entity.  
`GET /impact` returns consumers before deprecate.

### 5.6 Schema drift

Job compares `NATIVE_COLUMN` fields to Postgres `information_schema`.  
Statuses: `IN_SYNC`, `MISSING_IN_DB`, `MISSING_IN_METADATA`, `TYPE_MISMATCH`, `NULLABILITY_MISMATCH`.

### 5.7 Packages

Versioned bundles (`metadata_package`) install industry/tenant accelerators with checksum + signature hook.

### 5.8 Publish governance

Changeset → optional approval (`metadata_approval`) → publish version → artifact → outbox → cache invalidate.

### 5.9 Channel packs

Same entity, different form/list keys selected by channel + role.

### 5.10 Command / Query / Event descriptors

Catalog of `CreatePartner`, `ListPartners`, `bp.partner.created` for documentation, gateway generation, and AI tooling — **not** a replacement for code.

---

## 6. Ownership map

### Metadata owns

```text
dictionary, semantic layer, UI descriptors, validation AST,
FLS descriptors, extensions/overlays/packs,
changesets/approvals/publish artifacts,
dependency graph projections, drift reports,
command/query/event descriptors, catalog audit
```

### Other platforms own

| Platform | Owns |
|---|---|
| `pNN` / `bNN` | Physical tables, domain invariants, Alembic |
| `p03` | Setting values |
| `p01` | Real permission checks / sessions |
| `p06` | Translations |
| `p12` | Feature flag evaluation |
| Frontend | Rendering + a11y widgets |

---

## 7. Security contract

### Permissions

| Code | Use |
|---|---|
| `metadata.catalog.read` | Read published effective catalog |
| `metadata.catalog.manage` | Mutate system dictionary/UI |
| `metadata.extension.read` | Read tenant extensions |
| `metadata.extension.manage` | Tenant custom fields/overlays |
| `metadata.pack.install` | Install/uninstall packages |
| `metadata.publish` | Publish / rollback |
| `metadata.approve` | Approve changesets |
| `metadata.validate` | Validate payloads |
| `metadata.impact.read` | Impact / dependency APIs |
| `metadata.drift.read` | Drift reports |
| `metadata.security.manage` | FLS / classification policies |
| `metadata.audit.read` | Catalog audit trail |
| `metadata.*` | Wildcard |

### Trust rules

1. Never trust Metadata alone for authorization.  
2. Never execute non-allow-listed AST nodes.  
3. Never auto-migrate DDL from Metadata in production without human + Alembic.  
4. PII fields default to masked in list projections unless permission allows reveal.  
5. Draft lifecycle never served to `lifecycle=published` clients.

---

## 8. AST expression engine (safe)

Allowed node types (v1):

```text
LITERAL, FIELD_REF, CONTEXT_REF (tenant_id, roles, channel),
EQ, NE, IN, NOT, AND, OR,
GT, GTE, LT, LTE,
COALESCE, CONCAT (bounded),
LEN, IS_NULL, IS_EMPTY,
FEATURE_ENABLED (flag key)
```

Denied: SQL, Python, network, imports, loops unbounded, field writes.

Evaluation is deterministic and side-effect free. Hard timeout + max depth.

---

## 9. Caching strategy

| Layer | Key | Invalidate on |
|---|---|---|
| Effective UI pack | `(tenant, company, roles_hash, channel, entity, version)` | publish, overlay, pack install, feature flag |
| Published artifact | `(scope, version)` | new publish only (immutable) |
| ETag | checksum of effective payload | same |

Prefer Redis via future `p16_cache` policies; local process cache allowed with TTL.

---

## 10. Module layout (advanced)

```text
platforms/p05_metadata/
  module.py
  application/
    commands/…                 # mutate dictionary/UI/packs
    queries/…                  # read + effective resolve
    services/
      effective_resolver.py    # layered merge
      expression_engine.py     # safe AST
      publisher.py
      validator.py
      impact_analyzer.py
      drift_scanner.py
      package_installer.py
      redaction.py             # FLS masks
    permissions/catalog.py
    errors.py
  domain/
    aggregates/
    events/
    value_objects/             # DataType, SemanticType, AstNode
    policies/                  # key immutability, publish gates
  infrastructure/
    http/…
    persistence/models/
    persistence/rls.py
    messaging/outbox/
    adapters/postgres_drift.py
  tests/
    unit/expression/
    unit/resolver/
    unit/publish/
```

---

## 11. Integration rules

1. Business write APIs **should** call validate (or embed shared validator) for extensible entities.  
2. Search indexes fields with `is_searchable` + semantic analyzer hints.  
3. Reporting uses measures/dimensions from semantic layer.  
4. Feature flags gate field/action inclusion via resolver layer 80.  
5. Localization resolves all `*_label_key` / `*_message_key`.  
6. Gateways only — no ORM imports across platforms.  
7. Snapshot business documents may copy field labels at write time; keep `field_key` for joins.

---

## 12. Domain events

| Event | When |
|---|---|
| `metadata.catalog.published` | New immutable version |
| `metadata.catalog.rolled_back` | Rollback publish |
| `metadata.package.installed` / `uninstalled` | Pack lifecycle |
| `metadata.extension.changed` | Tenant custom field/overlay |
| `metadata.entity.deprecated` | Deprecation |
| `metadata.drift.detected` | Scanner findings |
| `metadata.changeset.submitted` / `approved` / `rejected` | Governance |
| `metadata.security.policy_changed` | FLS change |

Stream: `jesloterp:metadata:outbox`.

---

## 13. Data types & semantic types

### Physical data types

`STRING TEXT INTEGER DECIMAL BOOLEAN DATE DATETIME UUID ENUM JSON MONEY PERCENT EMAIL PHONE URL REF FILE_REF GEO_POINT BINARY_REF RICH_TEXT COMPOSITE`

### Example semantic types (seeded)

`GSTIN PAN IFSC IBAN EMAIL_OFFICIAL PHONE_E164 MONEY_INR MONEY_USD PERCENT_RATE STATUS_BADGE PARTNER_ROLE GEO_INDIA_PINCODE`

Semantic types reference default validation + mask + UI control.

---

## 14. Storage strategies

| Strategy | Use |
|---|---|
| `NATIVE_COLUMN` | System fields mapped to real columns |
| `JSONB_PATH` | Default tenant custom fields on aggregate |
| `EXTENSION_TABLE` | Sparse EAV / high cardinality extensions |
| `VIRTUAL` | Computed display fields |
| `EXTERNAL_REF` | Value mastered in another system (read-only projection) |

---

## 15. Build phases (advanced)

| Phase | Deliverable |
|---|---|
| P0 | Docs v2 (this set) |
| P1 | Skeleton, RLS, permissions, lookups |
| P2 | Dictionary + semantic types |
| P3 | UI descriptors + channel packs |
| P4 | AST engine + validate |
| P5 | Extensions + overlays + FLS |
| P6 | Changesets + approvals + publish/rollback |
| P7 | Packages + impact graph |
| P8 | Drift scanner + catalog audit |
| P9 | Effective resolver cache + internal mesh |
| P10 | Registry → **Live** |

---

## 16. Definition of Done (advanced platform)

- [x] Effective resolve deterministic with layer provenance on every field  
- [x] Published artifacts immutable + checksum verified on read  
- [x] AST engine rejects forbidden nodes in tests  
- [ ] Tenant cannot delete/alter system native storage  
- [x] Impact API blocks publish when `breaking=true` without override permission  
- [x] Drift job runnable; report API secured  
- [ ] FLS redaction helpers used by at least one consumer (BP list)  
- [x] UI pack ETag/caching works  
- [x] Package install transactional + rollback  
- [x] HTTP dictionary / overlay / publish persist+list on `AsyncSession` (empty catalog is `[]`, not memory fallback)  
- [x] FORCE RLS GUC on public HTTP access (`require_metadata_access`)  
- [x] Unit + contract tests green; no cross-schema FKs  

---

## 17. Anti-patterns

| Don’t | Do |
|---|---|
| Flat single-tenant catalog only | Layered effective resolve |
| `eval()` / SQL in formulas | Safe AST |
| Mutate published JSON | New version / rollback |
| Metadata auto-DDL in prod | Drift report + Alembic |
| Store setting values here | p03 |
| Hard-coded UI strings only | `label_key` + p06 |
| Skip impact on deprecate | Dependency gate |
| Trust FLS descriptors alone | IAM + server redaction |

---

## 18. Related documents

- Schema v2: [`METADATA_SCHEMA.md`](METADATA_SCHEMA.md)  
- API v2: [`METADATA_API.md`](METADATA_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
- Configuration boundary: [`../03_configuration/CONFIGURATION_GUIDE.md`](../03_configuration/CONFIGURATION_GUIDE.md)
