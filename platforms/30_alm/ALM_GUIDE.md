# JeslotERP ALM / Transport Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f30a0b1c2d3e` / `f30b1c2d3e4f`; not Production  
**Package:** `platforms.p30_alm`  
**PostgreSQL schema:** `alm`  
**Depends on:** `p01_identity`  
**Integrates with:** `p03_configuration`, `p05_metadata`, `p11_rules`, `p10_process` (transportable *artifacts*, not owned here)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`ALM_SCHEMA.md`](ALM_SCHEMA.md) · [`ALM_API.md`](ALM_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Packages, manifests, export/import, DEV→QA→PROD promotion, signature, rollback. AUD-006. |

---

## 1. Purpose (enterprise)

`p30_alm` is JeslotERP’s **application lifecycle / transport plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP CTS / gCTS / transport of copies**  
- **Dataverse solutions + patches**  
- **Salesforce change sets / 2GP / DevOps Center**  

Git remains source control. p30 is **not** a Git replacement. It manages **application/platform artifacts and transport semantics**.

### Owns

| Domain | Examples |
|---|---|
| Environments | DEV / QA / PROD records |
| Packages | Versioned transport units |
| Artifacts | Component refs (UUID + kind) |
| Manifests | Dependencies + versions |
| Signatures | Integrity checksum + signature blob |
| Export / import | Idempotent import |
| Promotion | Validated DEV→QA→PROD |
| Rollback metadata | Previous package version pointer |

### Does **not** own

| Concern | Owner |
|---|---|
| Git history / branches | Git |
| Per-platform in-app “changeset” catalogs | Those platforms (not landscape ALM) |
| Secret values inside packages | p03 refs only — never plaintext |
| Business document transport (invoices) | `bNN_*` |

### Critical split: Git vs ALM vs Configuration

| | **Git** | **p30 ALM** | **p03 Configuration** |
|---|---|---|---|
| Unit | Source file | Signed package of artifacts | Runtime setting value |
| Question | What is the code? | What moves across landscapes? | What is this tenant’s value? |
| Overlay | Branch | Tenant overlay vs system core | Scope layers |

---

## 2. Architectural position

```text
Authoring platforms (p03/p05/p10/p11)
        │  artifact refs (UUID + kind)
        ▼
   Package + manifest + signature
        │
        ├── export  → bundle JSON
        ├── import  → validate + idempotent apply
        └── promote DEV → QA → PROD (gates)
```

**Hard rules**

1. Dependency validation before import/promote.  
2. Package integrity (checksum) required.  
3. Import is idempotent on `(package_key, version)`.  
4. Rollback stores prior version — does not invent Git revert.  
5. No cross-schema FKs.  
6. PostgreSQL SoR.

---

## 3. Advanced design principles

1. **Landscape first** — environments are first-class.  
2. **System core vs tenant overlay** — `layer` on artifacts.  
3. **Signature required for PROD promote** — p28 HSM (`HSM-PKCS11`) only. Local HMAC is not a signature.  
4. **Promotion validation** — deps + signature + status.  
5. **Idempotent import.**  
6. **No silent overwrite** of ACTIVE PROD without promote gate.  
7. **Lean tables** (AUD-022).  
8. **CQRS HTTP.**

---

## 4. Core concepts

### 4.1 Lifecycle

```text
DEV → QA → PROD
```

### 4.2 Package states

```text
DRAFT → SEALED → EXPORTED → IMPORTED → PROMOTED → ROLLED_BACK
```

### 4.3 Manifest

```text
package_key, version, depends_on: [{ package_key, min_version }],
artifacts: [{ kind, artifact_ref, layer }]
```

### 4.4 Integrity

`checksum_sha256` over canonical JSON. Mismatch → `ALM_CHECKSUM_MISMATCH`.

---

## 5. Security

### Permissions

| Code | Use |
|---|---|
| `alm.package.read` | List/get |
| `alm.package.manage` | Create/seal |
| `alm.export` | Export |
| `alm.import` | Import |
| `alm.promote` | Promote |
| `alm.rollback` | Rollback |
| `alm.admin` | Environments |
| `alm.*` | Wildcard |

### RLS

FORCE RLS on tenant overlay packages. System-core packages may be global (`tenant_id` NULL) with `alm.admin`.

---

## 6. Module layout

```text
platforms/p30_alm/
  application/services/transport_service.py
  infrastructure/http/… persistence/…
```

---

## 7. Domain events

| Event | When |
|---|---|
| `alm.package.sealed` | Seal |
| `alm.package.imported` | Import |
| `alm.package.promoted` | Promote |
| `alm.package.rolled_back` | Rollback |

Stream: `jesloterp:alm:outbox`.

---

## 8. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs |
| P1 | Environments, packages, artifacts, RLS |
| P2 | Manifest + checksum |
| P3 | Export / import |
| P4 | Promote + rollback |
| P5 | Tests + **SoR-Live** |

---

## 9. Definition of Done (enterprise)

- [x] Dependency validation blocks import  
- [x] Checksum mismatch rejected  
- [x] Import of same version is idempotent  
- [x] PROD promote requires signature  
- [x] P30-LIVE-001: sign via p28 HSM (pytest/`PROVIDER_PENDING`; never `SHA256-HMAC-LOCAL`); multi-env deploy agent fail-closed  
- [x] Rollback records prior version  
- [x] Tenant RLS on overlay packages  
- [x] K28-33-001: list packages DB-first on `AsyncSession` (empty is `[]`)  
- [x] No Git replacement APIs  

---

## 10. Anti-patterns

| Don’t | Do |
|---|---|
| Treat per-platform changeset HTTP as CTS | Use p30 packages |
| Embed secret values in export | Store p03 secret *refs* |
| Promote unsigned to PROD | Require signature |
| Replace Git | Transport artifacts only |

---

## 11. Related documents

- Schema: [`ALM_SCHEMA.md`](ALM_SCHEMA.md)  
- API: [`ALM_API.md`](ALM_API.md)  
- RTM: [`ALM_RTM.md`](ALM_RTM.md)  
- Implementation record: [`ALM_IMPLEMENTATION_RECORD.md`](ALM_IMPLEMENTATION_RECORD.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
