# JeslotERP Privacy Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f31a0b1c2d3e` / `f31b1c2d3e4f`; not Production  
**Package:** `platforms.p31_privacy`  
**PostgreSQL schema:** `privacy`  
**Depends on:** `p01_identity`, `p19_audit` (audit primitives + legal hold *records*)  
**Integrates with:** all platforms via gateways (DSR orchestration)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`PRIVACY_SCHEMA.md`](PRIVACY_SCHEMA.md) · [`PRIVACY_API.md`](PRIVACY_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Subject, purpose, consent, DSR, erasure orchestration, legal-hold respect. |

---

## 1. Purpose (enterprise)

`p31_privacy` is JeslotERP’s **privacy orchestration plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP ILM / GDPR destruction** (orchestration, not per-table deletes)  
- **Dynamics / Salesforce privacy + DSR**  

Audit said: start in p19; split to p31 when DSR orchestration appears. This phase **creates p31** because cross-platform forget is a bounded context (prompt §14). p19 remains the audit/legal-hold *primitive*.

### Owns

| Domain | Examples |
|---|---|
| Data subject | Subject key + tenant |
| Purpose | Processing purposes |
| Consent | Grant / withdraw |
| Privacy policy | Versioned policy |
| DSR | ACCESS / ERASURE / RESTRICT |
| Operations | Orchestrated steps across platforms |
| Legal-hold *refs* | UUID to p19 hold — do not delete under hold |

### Does **not** own

| Concern | Owner |
|---|---|
| Append-only audit storage | `p19_audit` |
| Identity user row | `p01_identity` |
| Actual domain row mutation | Owning platform via port |

### Critical split: Audit vs Privacy

| | **p19 Audit** | **p31 Privacy** |
|---|---|---|
| Question | What happened? | What may we process / must we forget? |
| Delete | Never (immutable) | Orchestrates *lawful* erasure elsewhere |
| Legal hold | Hold records | p31 **refuses** erase when hold active |

**Rule:** Do not blindly delete. Respect legal hold, retention, audit, tenant isolation, authorization, referential integrity.

---

## 2. Architectural position

```text
DSR request
   │
   ▼
Authorize + tenant RLS
   │
   ▼
If ERASURE and legal hold → BLOCKED
   │
   ▼
Operation steps (gateway ports per platform)
   │
   ▼
Record operation + emit audit (p19)
```

v1 ships **orchestration records + deny-under-hold**. ERASURE calls named owning-platform adapters (`p04` soft-delete partner, `p01` soft-delete user, `p08` soft-delete media). p31 never `DELETE`s foreign schemas. Missing owner ref or non-`AsyncSession` → `SKIPPED`, not a fake erase.

---

## 3. Advanced design principles

1. **Every sensitive op is auditable.**  
2. **Fail-closed** on missing tenant.  
3. **Idempotent DSR** on `(subject_key, kind, idempotency_key)`.  
4. **ACCESS** returns a package **index** (locators), tenant-scoped. p31 ledger is always crawled; p04/p01/p08 add locators only when the owner row exists.  
5. **ERASURE** is requested, not immediately destroyed.  
6. **No cross-schema FKs.**  
7. **PostgreSQL SoR.**

---

## 4. Core concepts

### 4.1 DSR kinds

`ACCESS` · `ERASURE` · `RESTRICT`

### 4.2 DSR states

```text
OPEN → IN_PROGRESS → COMPLETED | BLOCKED | REJECTED
```

### 4.3 Consent

Purpose-scoped; withdraw does not erase history.

---

## 5. Security

### Permissions

| Code | Use |
|---|---|
| `privacy.subject.read` | Subjects |
| `privacy.consent.manage` | Consent |
| `privacy.dsr.manage` | Create/advance DSR |
| `privacy.dsr.read` | Read DSR |
| `privacy.admin` | Policy |
| `privacy.*` | Wildcard |

### RLS

FORCE RLS on all `prv_*` tenant tables.

---

## 6. Module layout

```text
platforms/p31_privacy/
  application/services/privacy_service.py
  application/ports/erase.py
  application/ports/corpus.py
  infrastructure/adapters/erase_owners.py  # p04 SoftDelete + p01 DeleteUser
  infrastructure/adapters/corpus_crawlers.py  # ACCESS index
  infrastructure/http/… persistence/…
```

---

## 7. Domain events

| Event | When |
|---|---|
| `privacy.dsr.opened` / `blocked` / `completed` | DSR |
| `privacy.consent.withdrawn` | Consent |

Stream: `jesloterp:privacy:outbox`.

---

## 8. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs |
| P1 | Subject, purpose, consent, RLS |
| P2 | DSR + hold check |
| P3 | Operations log |
| P4 | Tests + **SoR-Live** |

---

## 9. Definition of Done (enterprise)

- [x] Erasure blocked when legal hold ref is ACTIVE  
- [x] DSR ACCESS is tenant-scoped  
- [x] Consent withdraw is auditable  
- [x] No blind DELETE of domain rows in v1  
- [x] P31-LIVE-001: owning-platform erase adapters (p04/p01); p31 has no domain DELETE  
- [x] P31-LIVE-002: ACCESS corpus crawlers; index is not empty (p31 ledger + owner locators)  
- [x] FORCE RLS tested  
- [x] K28-33-001: list subjects DB-first on `AsyncSession` (empty is `[]`)  
- [x] p19 remains audit owner  

---

## 10. Anti-patterns

| Don’t | Do |
|---|---|
| `DELETE FROM bp_partner` inside p31 | Port + owning platform |
| Ignore legal hold | BLOCKED + audit |
| Store DSR results in-memory | PostgreSQL |

---

## 11. Related documents

- Schema: [`PRIVACY_SCHEMA.md`](PRIVACY_SCHEMA.md)  
- API: [`PRIVACY_API.md`](PRIVACY_API.md)  
- RTM: [`PRIVACY_RTM.md`](PRIVACY_RTM.md)  
- Implementation record: [`PRIVACY_IMPLEMENTATION_RECORD.md`](PRIVACY_IMPLEMENTATION_RECORD.md)  
- Audit: [`../19_audit/AUDIT_GUIDE.md`](../19_audit/AUDIT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
