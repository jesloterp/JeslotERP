# JeslotERP Number Series Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** **Live** — full backend, HTTP, ORM (67 tables), Alembic `c1d2e3f4a5b6`/`d2e3f4a5b6c7`, ModuleRegistry wired  
**Package:** `platforms.p07_number_series`  
**PostgreSQL schema:** `number_series`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`  
**Integrates with:** `p05_metadata` (entity/doc descriptors), `p06_localization` (labels), `p09_document` / transport docs (consumers), `p10_process` (approvals), `p11_rules` (dynamic segment rules), `p12_feature`, `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`NUMBER_SERIES_SCHEMA.md`](NUMBER_SERIES_SCHEMA.md) · [`NUMBER_SERIES_API.md`](NUMBER_SERIES_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise numbering control plane: series objects, segment composition, gapless vs buffered, reserve/allocate/commit/void, fiscal rollover, legal policies, thresholds, external intake, check digits, packages, governance. |

---

## 1. Purpose (enterprise)

`p07_number_series` is JeslotERP’s **document & master numbering control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP** — SNRO number range objects, NRIV intervals, subobjects (company/FY), external vs internal, buffering  
- **Microsoft Dynamics 365** — Number sequences, segment composition, continuous vs non-continuous, scope (shared/company/OU), preallocation  
- **Salesforce** — Autonumber fields + enterprise extensions for multi-company legal numbering  
- **Oracle E-Business / Fusion** — Document sequences, gapless accounting sequences, assignment rules  

It is **not** `SELECT nextval('seq')` wrapped in an API. It is the system that makes logistics ERP legally and operationally correct for:

1. **Bilty / LR / memo / challan / trip / invoice / payment** numbers  
2. **Company / branch / fiscal-year / document-type** scoped uniqueness  
3. **Pattern composition** (`{COMP}/{FY}/{BRANCH}/{SEQ:000000}`)  
4. **Gapless (continuous)** numbering where statute or audit demands no holes  
5. **High-throughput buffered** numbering for non-legal operational docs  
6. **Reserve → allocate → commit → void** lifecycle (draft bilty without burning legal invoice numbers incorrectly)  
7. **External / manual** series with format + uniqueness validation  
8. **Check digits**, charset rules, and collision detection  
9. **Threshold alerts** before range exhaustion  
10. **Fiscal rollover**, migration imports, gap scans, and immutable allocation ledger  

### Owns

| Domain | Examples |
|---|---|
| Series objects | `BILTY`, `INV_TAX`, `FRT_CHALLAN`, `BP_CODE` |
| Definitions & segments | Pattern, padding, constants, FY/MM tokens |
| Scopes & assignments | Tenant/company/branch/FY/doc-type binding |
| Intervals & counters | From/to, current, buffer pools |
| Allocation runtime | Peek, reserve, next, commit, void, recycle policy |
| Legal policies | Gapless, no-reuse, mandatory FY reset |
| External intake | User-supplied numbers with validation |
| Governance | Changesets, approvals, activate definition |
| Ops quality | Gap scan, reconcile, thresholds, usage stats |
| Packages | Country/legal numbering packs (India GST invoice) |

### Does **not** own

| Concern | Owner |
|---|---|
| Actual bilty/invoice business documents | Domain modules / `p09_document` |
| Fiscal calendar master | `p02_organization` (referenced by UUID) |
| Setting values (`default_series_policy`) | `p03_configuration` |
| Field “this entity has autonumber” UI meta | `p05_metadata` (points at `series_object_key`) |
| Translated labels for series names | `p06_localization` |
| Approval workflow engine | `p10_process` (hooks; local approval table for series def changes) |
| Blob storage of printed stationery | `p08_file_media` |

### Critical split: Series vs Document

| | **Number Series (p07)** | **Document / Domain** |
|---|---|---|
| Stores | Object, pattern, counter, issued value | Business payload (parties, amounts, lines) |
| Question | What is the next legal/op number? | What does this bilty contain? |
| Uniqueness | Enforced here per scope | References `allocation_id` / `formatted_number` |

---

## 2. Architectural position

```text
p01 ──► p02 ──► p03 ──► p07 number_series
                           │
     ┌──────────┬──────────┼──────────┬──────────┬──────────┐
     ▼          ▼          ▼          ▼          ▼          ▼
  Bilty/LR   Invoice    Challan    BP codes   Payments   Masters
  allocate   gapless    buffered   external   vouchers   codes
```

**Hard rules**

1. Domain modules **never** maintain their own `next_number` column logic.  
2. No cross-schema FKs — UUID refs to org company/branch/fiscal only.  
3. Allocation is **idempotent** (`Idempotency-Key` + business `reservation_token`).  
4. Gapless mode uses strict locking; buffered mode never claims gapless.  
5. Void does not “delete history”; it records void + policy-driven recycle.  
6. RLS fail-closed on tenant-scoped counters and allocations.

---

## 3. Advanced design principles

1. **Object-first** — `series_object_key` is the stable contract (`SALES_ORDER`, `TAX_INVOICE`).  
2. **Definition versioning** — pattern changes create new definition versions; activate via publish.  
3. **Segment composition** — numbers are built from ordered segments, not one opaque format string only (format string is derived/cached).  
4. **Scope dimensions** — SHARED / TENANT / COMPANY / BRANCH / FISCAL_YEAR / DOC_SUBTYPE.  
5. **Continuous vs non-continuous** — explicit modes; continuous ⇒ no buffer holes in committed set.  
6. **Two-phase issue** — `reserve` (optional) → `commit` on document post; or atomic `allocate`.  
7. **Peek never consumes** — dry-run / UI preview.  
8. **External series** — validate + register; do not auto-increment.  
9. **Legal policy binding** — GST tax invoice: gapless + no-reuse + FY scope.  
10. **Buffer pools** — pre-leased chunks for high TPS operational docs.  
11. **Check digit engines** — Luhn, Mod97, custom weighted — pluggable.  
12. **Exhaustion governance** — thresholds, auto-extend policy, hard stop.  
13. **Fiscal rollover jobs** — open next FY interval from rules.  
14. **Allocation ledger** — every issued number is queryable with provenance.  
15. **CQRS HTTP** — thin routers; allocator service is the concurrency hotspot.  
16. **Idempotent allocate** — same key returns same number; never double-burn.  
17. **Simulate** — format + next without write (admin).  
18. **Packages** — install India/AE numbering packs with checksum.

---

## 4. Core concepts

### 4.1 Series object

Stable product contract, e.g.:

| Object key | Typical use | Default mode |
|---|---|---|
| `BILTY` | Consignment / LR | Buffered or continuous (tenant policy) |
| `LOADING_SLIP` | Memo | Buffered |
| `FREIGHT_CHALLAN` | Challan | Continuous optional |
| `TRIP_SHEET` | Dispatch | Buffered |
| `TAX_INVOICE` | GST tax invoice | **Continuous / gapless** |
| `PROFORMA` | Quote | Buffered, reuse OK |
| `PAYMENT_VOUCHER` | Finance | Continuous often |
| `BP_CODE` | Partner code | External or internal |
| `VEHICLE_CODE` | Fleet | Internal |

### 4.2 Definition + segments

Example pattern for company-scoped bilty:

```text
Segments (ordered):
  1. CONSTANT      "BL"
  2. SEPARATOR     "/"
  3. COMPANY_CODE  from scope
  4. SEPARATOR     "/"
  5. FISCAL_YEAR   "2526" (YYYY compact) or "2025-26"
  6. SEPARATOR     "/"
  7. BRANCH_CODE   optional
  8. SEPARATOR     "/"
  9. SEQUENCE      numeric, pad 6, step 1
 10. CHECK_DIGIT   optional Mod97
→ BL/MH01/2526/NDL/000147-3
```

### 4.3 Scope resolution

Allocator inputs:

```text
series_object_key,
tenant_id, company_id?, branch_id?,
fiscal_year_id? | posting_date?,
doc_subtype?,
channel?,          # rare
manual_number?,    # external series
idempotency_key
```

Resolver picks **assignment** → **active definition** → **interval/counter** for that scope tuple.

### 4.4 Allocation modes

| Mode | Behavior | Use |
|---|---|---|
| `CONTINUOUS_GAPLESS` | Strict lock; no buffer; voids leave marked gaps or blocked reuse per policy | Tax invoice, statutory |
| `NON_CONTINUOUS_BUFFERED` | Buffer pool; possible unused holes on crash | Bilty high volume |
| `EXTERNAL_MANUAL` | Caller supplies number; validate format+unique | Legacy stationery |
| `HYBRID` | Prefer internal; allow manual override with permission | Migration period |

### 4.5 Lifecycle of a number

```text
PEEK (no write)
  → RESERVE (hold for N minutes / until document id)
  → COMMIT / ALLOCATE (permanent issue)
  → VOID (reason + actor) → optional RECYCLE (policy)
```

States on `ns_allocation`: `RESERVED` → `ISSUED` → `VOIDED` → (`RECYCLED` if allowed).

---

## 5. Concurrency & performance

### 5.1 Gapless path

- `SELECT … FOR UPDATE` on counter row (or advisory lock key).  
- Single-flight per scope interval.  
- No main-memory buffer.  
- Throughput intentionally lower; correctness first.

### 5.2 Buffered path

- `ns_buffer_pool` leases ranges to app instances (`1000–5000` typical).  
- Instance allocates in-memory; periodic checkpoint of high-water.  
- Crash may skip unused buffered values ⇒ **forbidden** under continuous policy.

### 5.3 Idempotency

```text
Idempotency-Key + tenant + object + scope hash
  → return prior allocation if completed
  → wait/reject if in-flight (configurable)
```

---

## 6. Legal & compliance policies

Attached via `ns_legal_policy` to object or assignment:

| Policy flag | Meaning |
|---|---|
| `require_gapless` | Mode must be continuous |
| `forbid_reuse` | Voids never recycle |
| `require_fiscal_scope` | FY dimension mandatory |
| `require_company_scope` | Company mandatory |
| `allow_manual_override` | Hybrid manual with permission |
| `max_void_rate_pct` | Alert if voids exceed threshold |
| `retain_allocation_years` | Ledger retention hint |

India GST tax invoice pack seeds `TAX_INVOICE` with gapless + no-reuse + company + FY.

---

## 7. Fiscal rollover

- Bind intervals to org fiscal year UUID.  
- `ns_period_reset_rule`: on FY open → create next interval (`from=1`, `to=…`).  
- Scheduler/ops triggers `rollover` command; emits `number_series.interval.opened`.  
- Cross-FY uniqueness is by scope; same sequence may restart when FY differs.

---

## 8. External / manual numbering

1. Client calls `allocate` with `manual_number`.  
2. Engine validates segments / regex / check digit.  
3. Uniqueness check in allocation ledger for scope.  
4. Registers as `ISSUED` with `source=EXTERNAL`.  
5. Does **not** advance internal counter unless policy `advance_on_external=true` (rare; usually false to avoid collisions — prefer disjoint ranges).

---

## 9. Governance

```text
DRAFT definition → changeset → approve → ACTIVATE (publish)
```

Activating a new pattern does not rewrite history; old allocations keep `formatted_number` + `definition_version_id`.

---

## 10. Security

### Permissions

| Code | Use |
|---|---|
| `number_series.catalog.read` | Read objects/definitions |
| `number_series.catalog.manage` | Manage objects/segments |
| `number_series.assign.manage` | Scope assignments |
| `number_series.allocate` | Next/reserve/commit (runtime) |
| `number_series.allocate.manual` | External/manual numbers |
| `number_series.void` | Void issued numbers |
| `number_series.recycle` | Recycle voids (if policy) |
| `number_series.publish` | Activate definitions |
| `number_series.approve` | Approve changesets |
| `number_series.pack.install` | Install packs |
| `number_series.ops.scan` | Gap/reconcile scans |
| `number_series.audit.read` | Audit / ledger query |
| `number_series.*` | Wildcard |

### RLS

FORCE RLS on tenant counters, buffers, allocations, reservations, voids.  
System seed objects readable; mutable with manage permission.

---

## 11. Module layout

```text
platforms/p07_number_series/
  application/
    services/
      scope_resolver.py
      segment_engine.py
      allocator.py          # hotspot
      buffer_manager.py
      gapless_lock.py
      check_digit.py
      legal_policy_guard.py
      rollover.py
      gap_scanner.py
      simulator.py
      package_installer.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/outbox/
  tests/unit/allocator/ segments/ gapless/
```

---

## 12. Integration rules

1. Document post handlers call **allocate/commit**, never local sequences.  
2. Store both `formatted_number` and `allocation_id` on the document.  
3. Draft documents: prefer `reserve` with TTL; release on discard.  
4. Metadata may declare `series_object_key` on entity; runtime still calls p07.  
5. Gateways only — no ORM imports across platforms.  
6. Configuration may hold defaults (`number_series.default_buffer_size`); engine reads via gateway.  
7. Never log full allocation ledger to world-readable logs (PII-adjacent partner codes rare; still sensitive commercially).

---

## 13. Domain events

| Event | When |
|---|---|
| `number_series.number.reserved` | Reserve |
| `number_series.number.issued` | Allocate/commit |
| `number_series.number.voided` | Void |
| `number_series.number.recycled` | Recycle |
| `number_series.definition.activated` | Publish |
| `number_series.interval.opened` / `exhausted` | Interval lifecycle |
| `number_series.threshold.breached` | Usage alert |
| `number_series.gap.detected` | Scan finding |
| `number_series.pack.installed` | Pack |

Stream: `jesloterp:number_series:outbox`.

---

## 14. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, seed objects, permissions |
| P2 | Definitions, segments, assignments, format engine |
| P3 | Atomic allocate + idempotency + ledger |
| P4 | Reserve/void + legal policies |
| P5 | Buffer pools + continuous mode locks |
| P6 | Fiscal rollover + thresholds |
| P7 | External intake + check digits + gap scan |
| P8 | Packages (India tax invoice) + governance |
| P9 | Registry → **Live** |

---

## 15. Definition of Done (enterprise)

- [ ] Gapless allocate under concurrency (multi-worker test) — no duplicates, no skips when continuous  
- [ ] Buffered allocate ≥ target TPS without duplicate formatted numbers  
- [ ] Idempotent allocate returns same number  
- [ ] Reserve TTL expiry releases hold without issuing  
- [ ] Tax invoice policy rejects buffer mode  
- [ ] FY rollover opens new interval  
- [ ] External number uniqueness enforced  
- [ ] Void + forbid_reuse blocks recycle  
- [ ] Threshold event emitted at configured %  
- [ ] Segment format deterministic golden tests  
- [ ] No cross-schema FKs  
- [ ] Tenant RLS on allocation ledger  

---

## 16. Anti-patterns

| Don’t | Do |
|---|---|
| `MAX(doc_no)+1` in domain tables | Call p07 allocator |
| One global sequence for all companies | Scope by company/FY |
| Claim gapless while using buffers | Separate modes |
| Delete allocation rows on void | Void status + audit |
| Change pattern in place silently | Version + activate |
| Allow client to invent invoice numbers unchecked | External validate path |
| Share buffer pools across gapless objects | Hard isolate |

---

## 17. Related documents

- Schema: [`NUMBER_SERIES_SCHEMA.md`](NUMBER_SERIES_SCHEMA.md)  
- API: [`NUMBER_SERIES_API.md`](NUMBER_SERIES_API.md)  
- Organization fiscal: [`../02_organization/ORGANIZATION_GUIDE.md`](../02_organization/ORGANIZATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
