# JeslotERP Document Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — DIR/versions/libraries Postgres-first when session is AsyncSession. Empty list is `[]`. Not ArchiveLink. Not Production.  
**Package:** `platforms.p09_document`  
**PostgreSQL schema:** `document`  
**Depends on:** `p07_number_series`, `p08_file_media` (registry); also `p01_identity`, `p02_organization` in practice  
**Integrates with:** `p03_configuration`, `p05_metadata`, `p06_localization`, `p04_business_partner`, `p10_process` (approvals), `p11_rules`, `p12_feature`, `p15_notification`, `p18_search`, `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`DOCUMENT_SCHEMA.md`](DOCUMENT_SCHEMA.md) · [`DOCUMENT_API.md`](DOCUMENT_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise DMS: DIR-class records, types/status network, check-in/out, versions/revisions, libraries, object links, templates/merge, renditions, ACL, retention/legal hold, controlled copy, e-sign hooks, compounds, governance packs. |
| 1.1 | 2026-09-12 | TASK-SOR-011: durable DIR/versions/libraries; empty list is `[]`; RLS on `require_document_access`. |

---

## 1. Purpose (enterprise)

`p09_document` is JeslotERP’s **document management (DMS) control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP DMS / Document Info Record (DIR)** — document types, status network, versions, object links, classification  
- **Microsoft Dynamics 365** — document management, SharePoint-backed libraries, templates on records  
- **Salesforce** — ContentDocument / ContentVersion / Libraries / sharing  
- **Enterprise ECM patterns** — check-in/out, renditions, retention, controlled distribution  

It is **not** a file upload table. **Bytes live in `p08_file_media`.** This platform owns the **business document identity**, lifecycle, and relationships.

It makes logistics ERP correct for:

1. **Versioned business documents** (contracts, rate cards, KYC packs, scanned LRs, signed PODs as DMS records)  
2. **Document types + status networks** (DRAFT → IN_REVIEW → RELEASED → OBSOLETE)  
3. **Check-out / check-in** with exclusive locks  
4. **Revisions & minor/major versions** bound to `media_id` content  
5. **Object links** — document ↔ bilty / BP / vehicle / invoice / employee  
6. **Libraries / cabinets / folders** for governed collections  
7. **Templates + mail-merge** into generated PDFs (output media via p08)  
8. **Renditions** (DOCX→PDF, watermarked controlled copy)  
9. **Numbering** via `p07_number_series` (`DOC_*` objects)  
10. **ACL, retention, legal hold, e-sign hooks, distribution, compound structures**

### Owns

| Domain | Examples |
|---|---|
| Document info records | Logical document identity |
| Types & status network | CONTRACT, POD_SCAN, KYC_PACK + statuses |
| Versions / revisions | v1.0, v1.1 → media_id |
| Check-out locks | Who edits, since when |
| Libraries & folders | Cabinet structure |
| Object links | Links to domain entities |
| Templates & merge | Merge fields, generate jobs |
| Renditions | PDF preview, controlled copy |
| Classification / confidentiality | INTERNAL, CONFIDENTIAL, … |
| ACL & sharing | Princip / roles / link shares |
| Retention & legal hold | Doc-level (coordinates media) |
| Distribution & controlled copy | Watermarked releases |
| E-sign envelopes | Hook to providers (not crypto keys) |
| Compounds | Parent/child document structures |
| Governance | Changesets, packs, approvals |

### Does **not** own

| Concern | Owner |
|---|---|
| Raw bytes / AV scan / CDN | `p08_file_media` |
| Sequence allocation | `p07_number_series` |
| Transport bilty business payload | Domain transport module |
| BPM approval engine (generic) | `p10_process` (status transitions may call it) |
| Full-text index | `p18_search` |
| Notification delivery | `p15_notification` |
| Setting values | `p03_configuration` |

### Critical split: Media vs Document vs Domain

| Layer | Owns | Example |
|---|---|---|
| **p08 media** | Bytes, scan, variants | `media_id=M1` PDF bytes |
| **p09 document** | DIR, version → M1, status RELEASED | `DOC-2026-00421` Contract |
| **Domain** | Business transaction | Bilty `B-100` with amount/parties — may **link** to documents |

---

## 2. Architectural position

```text
p07 number_series ──► document number on create
p08 file_media    ──► version content (media_id)
         │
         ▼
   p09 document (DMS)
         │
   ┌─────┼──────┬──────────┬─────────┐
   ▼     ▼      ▼          ▼         ▼
  BP   Bilty  Invoice   Vehicle   Process
 links links  links     links     approve
```

**Hard rules**

1. Every contentful version **must** reference `media_id` (or be TEMPLATE_ONLY metadata).  
2. No cross-schema FKs — UUID refs only.  
3. Check-out is mandatory before replacing RELEASED content when type policy requires it.  
4. RELEASED versions are immutable; new change ⇒ new version.  
5. Legal hold on document propagates hold request to linked media via gateway.  
6. RLS fail-closed on tenant documents.

---

## 3. Advanced design principles

1. **DIR-first** — stable `document_id` + human `document_number`.  
2. **Type-driven behavior** — policies hang off document type.  
3. **Status network** — allowed transitions, not free-form strings.  
4. **Version immutability** — content+metadata snapshot per version.  
5. **Check-out exclusivity** — one writer; break-lock is audited admin op.  
6. **Object links many-to-many** — one doc many entities; one entity many docs.  
7. **Template ≠ instance** — generate creates new DIR or new version.  
8. **Rendition pipeline** — async jobs; store result media_ids.  
9. **Controlled copy** — watermark + purpose + recipient log.  
10. **Number series integration** — allocate on create when type requires.  
11. **ACL layered** — library default → document ACL → link grants.  
12. **Retention schedules** — event-based (from RELEASED / from contract end).  
13. **Compound documents** — structure trees (KYC pack = many children).  
14. **E-sign is orchestration** — envelope state here; provider secrets in p03.  
15. **CQRS HTTP** — thin routers; domain services for lock/version.  
16. **Idempotent generate / check-in**.  
17. **Search events** — emit on release for p18.  
18. **Packages** — India KYC pack / contract type pack.

---

## 4. Core concepts

### 4.1 Document Info Record (DIR)

```text
document_id, document_number, type_key, title,
status_code, current_version_id, library_id?,
classification, company_id, owner_user_id, tenant_id
```

### 4.2 Version model

| Field | Notes |
|---|---|
| `version_label` | `1.0`, `1.1`, `2.0` |
| `revision` | Integer monotonic |
| `media_id` | Content in p08 |
| `change_comment` | Check-in note |
| `content_role` | PRIMARY, ATTACHMENT_SLOT, RENDITION_SOURCE |
| `is_current` | Points from DIR |

Major/minor rules configurable per type (`bump_major_on_release`).

### 4.3 Status network (example)

```text
DRAFT ──► IN_REVIEW ──► APPROVED ──► RELEASED ──► OBSOLETE
              │                         │
              └──────── REJECTED ───────┘
LEGAL_HOLD can overlay without changing network code
```

Transitions carry permission + optional p10 process key.

### 4.4 Check-out

```text
AVAILABLE → CHECKED_OUT (by user, expires_at optional)
check-in uploads/links new media → new version → DRAFT/IN_REVIEW
undo check-out discards lock without version
break-lock (admin) audited
```

### 4.5 Object links

```text
document_id ↔ entity_type + entity_id + link_role
Roles: PRIMARY, SUPPORTING, EVIDENCE, SIGNED_COPY, ANNEX
```

### 4.6 Libraries

Cabinet → Library → Folder tree. Defaults for ACL, retention, allowed types.

### 4.7 Templates

- Template DIR or `doc_template` definition with merge field map  
- `generate` resolves fields from entity gateway → render → p08 media → new DIR/version  
- Merge field allow-list (no arbitrary code)

### 4.8 Controlled copy / distribution

When releasing externally:

1. Create rendition with watermark profile  
2. Log recipient, purpose, expiry  
3. Optional share via media share or doc distribution grant  

### 4.9 Compound documents

Parent DIR (e.g. `KYC_PACK`) with child DIRs ordered; release rules (all children RELEASED).

---

## 5. Integration with p07 / p08

| Need | Call |
|---|---|
| New document number | `p07` allocate `DOC_CONTRACT` / type-mapped object |
| Store PDF/DOCX | `p08` upload → `media_id` |
| Version content | `doc_version.media_id = …` |
| Preview thumb | p08 variant; doc API may proxy metadata |
| Delete content | Soft-delete doc; media delete per policy gateway |
| Legal hold | Doc hold + `media.legal_hold` gateway |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `document.read` | Read meta/content if ACL |
| `document.create` | Create DIR |
| `document.checkout` | Check-out / check-in |
| `document.release` | Transition to RELEASED |
| `document.obsolete` | Obsolete |
| `document.delete` | Soft delete |
| `document.acl.manage` | ACL |
| `document.template.manage` | Templates |
| `document.template.generate` | Generate from template |
| `document.admin` | Types, status network, break-lock |
| `document.legal_hold` | Holds |
| `document.distribute` | Controlled copies |
| `document.esign` | E-sign envelopes |
| `document.audit.read` | Audit |
| `document.*` | Wildcard |

### RLS

FORCE RLS on tenant documents, versions, links, checkouts, ACL.

---

## 7. Module layout

```text
platforms/p09_document/
  application/
    services/
      document_factory.py
      versioning.py
      checkout_lock.py
      status_network.py
      object_link.py
      template_merge.py
      rendition.py
      controlled_copy.py
      esign_orchestrator.py
      retention.py
      compound.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/outbox/
    gateways/ number_series.py media.py process.py
  tests/unit/versioning/ checkout/ status/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `document.created` / `updated` | DIR |
| `document.checked_out` / `checked_in` / `checkout_undone` | Locks |
| `document.version.created` | New version |
| `document.status.changed` | Network transition |
| `document.released` / `obsoleted` | Milestones |
| `document.link.added` / `removed` | Object links |
| `document.generated` | From template |
| `document.rendition.ready` | Async |
| `document.distributed` | Controlled copy |
| `document.legal_hold.applied` / `released` | Hold |
| `document.esign.completed` / `declined` | E-sign |
| `document.deleted` | Soft delete |

Stream: `jesloterp:document:outbox`.

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, types, status network seeds |
| P2 | DIR create + p07 number + version↔media |
| P3 | Check-out/in + immutability |
| P4 | Object links + libraries/folders |
| P5 | ACL + release transitions |
| P6 | Templates + generate |
| P7 | Renditions + controlled copy |
| P8 | Retention/legal hold + compounds |
| P9 | E-sign hooks + packs |
| P10 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [x] Version content immutable after RELEASED  
- [x] Concurrent check-out denied for second user  
- [x] Status transition matrix enforced  
- [x] Object link query by entity returns docs  
- [x] Template generate produces media + version idempotently  
- [x] Legal hold blocks delete and obsolete  
- [x] Document number from p07 when required  
- [x] No raw bytes stored in `document` schema  
- [x] Tenant RLS GUCs on HTTP (`require_document_access`)  
- [x] DIR/versions/libraries persist on `AsyncSession` (empty catalog is `[]`)  
- [x] No cross-schema FKs  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Store PDF bytes in document tables | `media_id` → p08 |
| Overwrite RELEASED version in place | New version |
| Free-text status without network | Status transition API |
| Duplicate upload logic in p09 | Call p08 |
| Local `MAX+1` document numbers | p07 allocate |
| Treat bilty row as DMS | Link bilty ↔ document |

---

## 12. Related documents

- Schema: [`DOCUMENT_SCHEMA.md`](DOCUMENT_SCHEMA.md)  
- API: [`DOCUMENT_API.md`](DOCUMENT_API.md)  
- Media: [`../08_file_media/FILE_MEDIA_GUIDE.md`](../08_file_media/FILE_MEDIA_GUIDE.md)  
- Number series: [`../07_number_series/NUMBER_SERIES_GUIDE.md`](../07_number_series/NUMBER_SERIES_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
