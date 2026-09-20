# JeslotERP Audit Platform — Developer Integration Guide

**Version:** 1.3  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — event ingest/query Postgres-first; SIEM HTTP is a fail-closed port (`PROVIDER_PENDING`). Not Production.  
**Package:** `platforms.p19_audit`  
**PostgreSQL schema:** `audit`  
**Depends on:** `p01_identity`, `p02_organization`, `p13_event_bus`  
**Integrates with:** all platforms (emitters), `p03_configuration`, `p08_file_media` (export artifacts), `p12_feature`, `p14_messaging`, `p15_notification` (alert), `p16_cache`, `p17_scheduler` (retention), `p18_search` (optional find), `p20_logging` (ops logs ≠ audit)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`AUDIT_SCHEMA.md`](AUDIT_SCHEMA.md) · [`AUDIT_API.md`](AUDIT_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise audit plane: immutable events, hash-chain integrity, actors/actions catalog, field-level diffs, retention/legal hold, redaction views, ingest from event-bus, export, SIEM hooks, access-to-audit auditing. |
| 1.1 | 2026-09-12 | TASK-SOR-017: HTTP `POST /events` append-only on Postgres; query fetch is DB-first. |
| 1.2 | 2026-09-12 | TASK-SOR-024: SIEM HTTP port fail-closed; pytest stays STUB; `POST /siem/endpoints/{key}/test-connection`. |
| 1.3 | 2026-09-12 | HYG-019 pointer to TASK-SOR-024. Stay in p19. No Splunk/Sentinel client. |

---

## 1. Purpose (enterprise)

`p19_audit` is JeslotERP’s **immutable compliance audit control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Change Documents / AIS / ILM audit patterns** — who changed what, when, with retention  
- **Microsoft Dynamics 365 auditing** — entity/field audit, user access, retention policies  
- **Salesforce Setup Audit Trail / Field History / Event Monitoring** — admin & data change trails  
- **Banking / SOX / GST e-audit expectations** — tamper-evident, queryable, exportable  

It is **not** application debug logging. It is the system that makes ERP accountability correct for:

1. **Append-only audit events** — no update/delete of facts  
2. **Who / what / when / where / why** — actor, action, entity, correlation  
3. **Before/after field diffs** (policy-driven, PII-aware)  
4. **Tamper evidence** — hash chain / batch seals  
5. **Retention & legal hold** — ILM-style keep/purge rules  
6. **Ingest** from p13 domain events + explicit audit write API  
7. **Query & export** for compliance officers  
8. **Sensitive-action alerts**  
9. **Audit-of-audit** — reading/exporting audit is itself audited  
10. **SIEM / webhook delivery** of signed batches — live HTTP is a port; without a configured vendor webhook, ping/forward are `PROVIDER_PENDING` (never invented DELIVERED)  

### Owns

| Domain | Examples |
|---|---|
| Action & object catalogs | LOGIN, UPDATE, APPROVE, ALLOCATE_NUMBER |
| Audit events | Immutable records |
| Field changes | Diff lines |
| Integrity | Hash chains, seals |
| Retention / holds | Policies, purge jobs |
| Redaction | View policies for PII |
| Ingest | Event bindings, writers |
| Export | Cases, artifacts |
| Alerts | Rules on actions |
| Access audit | Who queried audit |

### Does **not** own

| Concern | Owner |
|---|---|
| Debug/trace logs | `p20_logging` |
| Metrics/APM | `p21_monitoring` |
| AuthN sessions themselves | `p01_identity` (emits audit) |
| Business source data | Domain modules |
| Generic event contracts | `p13_event_bus` (transport; audit is sink) |

### Critical split: Audit vs Logging vs Events

| | **Audit (p19)** | **Logging (p20)** | **Event Bus (p13)** |
|---|---|---|---|
| Purpose | Compliance accountability | Ops diagnostics | System integration |
| Mutability | Immutable | Rotatable | Durable but not legal SoR |
| Content | Actor+entity+action | Stack traces, levels | Business facts |
| Retention | Legal policy years | Days/weeks typical | Shorter product retention |

**Rule:** If a regulator asks “who changed this GST invoice number?”, the answer comes from **p19**, not log grepping.

---

## 2. Architectural position

```text
Platform command / p13 event
           │
           ▼
    audit ingest (validate + redact policy)
           │
           ├─ append event (+ field diffs)
           ├─ hash-link to previous
           └─ optional alert / search index / SIEM batch
```

**Hard rules**

1. **No UPDATE/DELETE** on audit event rows (DB roles + app).  
2. Purge only via retention job after hold checks — never ad-hoc SQL.  
3. Writers are allow-listed services; end users don’t forge arbitrary audits easily.  
4. Cross-schema: UUID refs only.  
5. RLS on tenant events for query; integrity chain may be tenant-scoped or global-sealed batches.  
6. Secrets never stored in before/after (mask).

---

## 3. Advanced design principles

1. **Append-only** with DB grants denying mutation.  
2. **Canonical action verbs** — catalog, not free text soup.  
3. **Object types** aligned to metadata entity keys where possible.  
4. **Correlation** — `request_id`, `correlation_id`, `causation_id`, `session_id`.  
5. **Field-level audit** selective by policy (not every column blindly).  
6. **Hash chain** — `prev_hash` + `event_hash` per stream (tenant or shard).  
7. **Batch seals** — periodic Merkle/seal for export verification.  
8. **Redacted projections** for normal investigators vs break-glass.  
9. **Legal hold** blocks purge.  
10. **Ingest idempotency** — `(source, source_event_id)` unique.  
11. **Time authoritative** — server `occurred_at`; client time secondary.  
12. **Actor types** — USER, SYSTEM, SERVICE, SUPPORT_BREAK_GLASS.  
13. **Export cases** — packaged evidence with checksum.  
14. **Alert rules** — e.g. mass delete, permission grant, kill switch.  
15. **Query cost controls** — indexed access paths, export async.  
16. **CQRS HTTP** — write ingest vs read investigation.  
17. **Search optional** — p18 index of non-PII audit summaries.  
18. **Packs** — seed action catalogs per platform.

---

## 4. Core concepts

### 4.1 Audit event

```text
event_id, occurred_at, tenant_id, company_id?,
actor_type, actor_id, actor_display,
action_key, object_type, object_id, object_display?,
outcome SUCCESS|FAIL|DENIED,
correlation_id, request_id, session_id, ip, user_agent,
source_platform, source_event_id?,
prev_hash, event_hash,
data_summary{},   # non-sensitive
```

### 4.2 Field change

```text
event_id, field_key, old_value_redacted, new_value_redacted, value_type
```

### 4.3 Integrity stream

Per `(tenant_id, stream_key)` ordered chain; seal every N events or time window.

### 4.4 Retention

| Class | Example |
|---|---|
| AUTH | 2–7 years |
| FINANCIAL | 7–10 years (jurisdiction pack) |
| OPERATIONAL | 1–3 years |
| SECURITY | longer |

---

## 5. Ingest paths

1. **Explicit** `POST /audit/events` from platform services (trusted)  
2. **Event-bus binding** — map `document.released.v1` → action `DOCUMENT_RELEASE`  
3. **CDC-lite hooks** — optional; prefer domain emit for meaning  

Idempotent on source ids.

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `audit.write` | Service ingest |
| `audit.read` | Query redacted trail |
| `audit.read.sensitive` | Unredacted / break-glass |
| `audit.export` | Create export cases |
| `audit.hold` | Legal holds |
| `audit.retention.manage` | Policies |
| `audit.alert.manage` | Alert rules |
| `audit.admin` | Seals, packs, bindings |
| `audit.access.read` | See who accessed audit |
| `audit.*` | Wildcard |

### RLS

FORCE RLS by `tenant_id` on events for normal readers.  
Break-glass elevates with reason + audit-of-audit entry.

---

## 7. Module layout

```text
platforms/p19_audit/
  application/
    services/
      ingest.py
      hasher.py
      redaction.py
      query.py
      retention.py
      legal_hold.py
      export.py
      alert.py
      seal.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/ siem/
  tests/unit/hash/ redact/ retention/ ingest/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `audit.event.appended` | (meta; careful volume — sample) |
| `audit.seal.created` | Integrity |
| `audit.hold.applied` / `released` | Legal |
| `audit.export.completed` | Export |
| `audit.alert.fired` | Alerts |
| `audit.purge.completed` | Retention |
| `audit.access.recorded` | Audit-of-audit |

Prefer low-volume meta events; don’t storm bus with every append.

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, catalogs, permissions |
| P2 | Ingest + immutable store + query |
| P3 | Field diffs + redaction |
| P4 | Hash chain + seals |
| P5 | Event-bus bindings |
| P6 | Retention + legal hold |
| P7 | Export cases + alerts |
| P8 | SIEM webhook + access audit |
| P9 | Packs (India financial retention) |
| P10 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] DB role cannot UPDATE/DELETE audit_event *(ORM insert-only; live DB GRANT not soak-tested)*  
- [x] Hash chain verify detects tamper in test  
- [x] Idempotent ingest on source_event_id  
- [ ] Redaction hides PAN/GSTIN patterns per policy *(password/token mask exists; full PAN/GSTIN pack later)*  
- [x] Legal hold blocks purge  
- [x] Export artifact checksum verified  
- [x] Reading sensitive audit creates access event  
- [x] Tenant RLS GUCs on query persist/fetch (`apply_aud_rls`)  
- [x] No secrets in diffs  
- [x] No cross-schema FKs  
- [x] Live SIEM HTTP is a fail-closed port (`PROVIDER_PENDING`); pytest stays STUB  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Log-only “audit” in p20 | Immutable p19 events |
| Allow UPDATE on audit rows | Append-only |
| Store raw passwords/tokens in diffs | Mask / omit |
| Unbounded sync export in HTTP | Async export case |
| Skip actor on system jobs | actor_type=SERVICE |
| Delete old rows manually | Retention job only |

---

## 12. Related documents

- Schema: [`AUDIT_SCHEMA.md`](AUDIT_SCHEMA.md)  
- API: [`AUDIT_API.md`](AUDIT_API.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_GUIDE.md`](../13_event_bus/EVENT_BUS_GUIDE.md)  
- Logging: [`../20_logging/LOGGING_GUIDE.md`](../20_logging/LOGGING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
