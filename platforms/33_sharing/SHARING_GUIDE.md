# JeslotERP Sharing Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f33a0b1c2d3e` / `f33b1c2d3e4f`; not Production  
**Package:** `platforms.p33_sharing`  
**PostgreSQL schema:** `sharing`  
**Depends on:** `p01_identity` (who the user is + RBAC), `p02_organization` (tenant/company exist)  
**Integrates with:** `p04_business_partner` GET / list / satellites / validate-for-use; `p08_file_media` objects; `p09_document` documents; `p10_process` instances (when grants exist); future `bNN_*`; p01 does **not** expand into this BC  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`SHARING_SCHEMA.md`](SHARING_SCHEMA.md) · [`SHARING_API.md`](SHARING_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Owner, team, role hierarchy, sharing rules, manual share, record ACL. AUD-007. |
| **1.1** | **2026-09-12** | P33-LIVE-001: HTTP evaluate reads grants from Postgres. |
| **1.2** | **2026-09-12** | P33-LIVE-002: p04 list partners share-filters. |
| **1.3** | **2026-09-12** | P33-LIVE-003: more p04 domain reads evaluate. |
| **1.4** | **2026-09-12** | PROD-KERN-033: p08/p09/p10 list+GET evaluate (skip if no grants). |

---

## 1. Purpose (enterprise)

`p33_sharing` is JeslotERP’s **record-level access plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **Salesforce OWD / role hierarchy / sharing rules / manual shares / implicit parent**  
- **Dynamics BU / team / owner**  
- **SAP org-level auth objects (V_FACT-class) as a BC — not IAM**  

### Owns

| Domain | Examples |
|---|---|
| Teams | Team + members |
| Role hierarchy | Parent/child role codes (UUID refs to p01 roles) |
| Sharing rules | Criteria-lite: owner/team/role → grant |
| Manual shares | Explicit grant/revoke |
| Record ACL | Effective grants on `(resource_type, resource_id)` |
| Access evaluation | Can user access this resource? |

### Does **not** own

| Concern | Owner |
|---|---|
| Authentication / RBAC permission codes | `p01_identity` |
| Tenant / company master | `p02_organization` |
| Tenant RLS (row isolation) | Each platform’s schema |
| Sales-order / CRM objects | `bNN_*` |

### Critical split: Tenant RLS vs Record sharing

| | **Tenant RLS** | **p33 Sharing** |
|---|---|---|
| Question | Can this session see tenant A’s rows at all? | Can this user see *this* record? |
| Mechanism | Postgres GUC `app.tenant_id` | ACL + rules + team + hierarchy |
| Owner | Each platform schema | p33 |

**p01 answers:** Can the user access the organization/tenant?  
**p33 answers:** Can the user access this specific resource/record?

Do **not** fake sharing with tenant RLS.

---

## 2. Architectural position

```text
evaluate(user, resource_type, resource_id)
        │
        ├── owner match?
        ├── team membership?
        ├── role hierarchy implicit?
        ├── sharing rule?
        └── manual grant?
        │
        ▼
   { allowed, reason, grant_ids }
```

**Hard rules**

1. Generic `resource_type` + UUID `resource_id` — no sales-order tables.  
2. PostgreSQL SoR for grants/teams/rules.  
3. No cross-schema FKs to domain tables.  
4. Fail-closed when the resource has any grant: no matching grant → deny (unless owner). No grants on the resource → skip (tenant RLS only) so existing p04 reads stay compatible.  
5. Concurrent identical grants do not duplicate (unique constraint).

---

## 3. Advanced design principles

1. **Owner is a grant reason** (`OWNER`).  
2. **Manual share** (`MANUAL`) revoke-able.  
3. **Team share** (`TEAM`).  
4. **Rule share** (`RULE`).  
5. **Implicit hierarchy** (`HIERARCHY`) — ancestor roles.  
6. **Evaluate is the product API.**  
7. **Idempotent grant.**  
8. **Reasons are auditable.**  

---

## 4. Core concepts

### 4.1 Access levels

`READ` · `WRITE` · `FULL`

### 4.2 Grant unique

`(tenant_id, resource_type, resource_id, grantee_kind, grantee_id, reason)` unique where not deleted.

### 4.3 Grantee kinds

`USER` · `TEAM` · `ROLE`

---

## 5. Security

### Permissions

| Code | Use |
|---|---|
| `sharing.team.manage` | Teams |
| `sharing.rule.manage` | Rules |
| `sharing.grant` | Manual share |
| `sharing.revoke` | Revoke |
| `sharing.evaluate` | Evaluate |
| `sharing.admin` | Hierarchy |
| `sharing.*` | Wildcard |

### RLS

FORCE RLS on all `shr_*` tenant tables.

---

## 6. Module layout

```text
platforms/p33_sharing/
  application/services/sharing_service.py
  infrastructure/http/… persistence/…
```

---

## 7. Domain events

| Event | When |
|---|---|
| `sharing.grant.created` / `revoked` | ACL |
| `sharing.team.changed` | Team |
| `sharing.rule.activated` | Rule |

Stream: `jesloterp:sharing:outbox`.

---

## 8. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs |
| P1 | Teams, grants, RLS |
| P2 | Evaluate owner/manual/team |
| P3 | Role hierarchy + rules |
| P4 | Concurrency + tests + **SoR-Live** |

---

## 9. Definition of Done (enterprise)

- [x] Owner can access; stranger cannot  
- [x] Manual grant/revoke  
- [x] Team membership grants access  
- [x] Hierarchy ancestor can access  
- [x] Duplicate grant is idempotent  
- [x] Tenant RLS on grants  
- [x] P33-LIVE-001: evaluate reads grants from Postgres on `AsyncSession` (empty list denies; no memory fallback)  
- [x] P33-LIVE-002: p04 list partners share-filters (skip if no grants; drop stranger rows)  
- [x] P33-LIVE-003: p04 public partner-scoped reads evaluate (GET + validate-for-use); not CRM  
- [x] PROD-KERN-033: p08 objects / p09 documents / p10 instances list+GET evaluate (skip if no grants)  
- [x] K28-33-001: list teams DB-first on `AsyncSession` (empty is `[]`)  
- [x] No CRM/sales tables  

---

## 10. Anti-patterns

| Don’t | Do |
|---|---|
| `if tenant_id == user.tenant_id: allow record` | Evaluate p33 |
| `sales_order_share` table | Generic resource id |
| Put sharing inside p01 | Keep the BC |

---

## 11. Related documents

- Schema: [`SHARING_SCHEMA.md`](SHARING_SCHEMA.md)  
- API: [`SHARING_API.md`](SHARING_API.md)  
- RTM: [`SHARING_RTM.md`](SHARING_RTM.md)  
- Implementation record: [`SHARING_IMPLEMENTATION_RECORD.md`](SHARING_IMPLEMENTATION_RECORD.md)  
- Identity: [`../01_identity/IDENTITY_GUIDE.md`](../01_identity/IDENTITY_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
