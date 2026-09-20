# JeslotERP Sharing Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f33a0b1c2d3e` / `f33b1c2d3e4f`  
**Package:** `platforms.p33_sharing`  
**PostgreSQL schema:** `sharing`  
**Companion:** [`SHARING_GUIDE.md`](SHARING_GUIDE.md) · [`SHARING_API.md`](SHARING_API.md)

> Runtime models: `platforms/p33_sharing/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `sharing` (never `p33`) |
| Tables | `shr_*` |
| Cross-schema | `resource_id` / `user_ref` UUIDs — no FK |
| RLS | FORCE on all tenant tables |

---

## 2. Complete table inventory (**6 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `shr_team` | Team |
| 2 | `shr_team_member` | Membership |
| 3 | `shr_role_hierarchy` | Parent/child role refs |
| 4 | `shr_sharing_rule` | Criteria-lite rule |
| 5 | `shr_grant` | ACL row |
| 6 | `shr_access_eval` | Evaluation log (optional sample) |
| 7 | `shr_outbox` | Outbox |
| 8 | `shr_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| Grantee kind | `USER`, `TEAM`, `ROLE` |
| Reason | `OWNER`, `MANUAL`, `TEAM`, `RULE`, `HIERARCHY` |
| Access | `READ`, `WRITE`, `FULL` |

---

## 4. Detailed tables

### 4.1 `shr_team`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `team_key` | VARCHAR(80) | Unique per tenant |
| `name` | VARCHAR(120) | |

### 4.2 `shr_team_member`

| Column | Type | Notes |
|---|---|---|
| `team_id` | UUID | |
| `user_ref` | UUID | p01 user — no FK |

**Unique:** `(team_id, user_ref)` where not deleted.

### 4.3 `shr_role_hierarchy`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `parent_role_ref` | UUID | p01 role |
| `child_role_ref` | UUID | |

### 4.4 `shr_sharing_rule`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `rule_key` | VARCHAR(80) | |
| `resource_type` | VARCHAR(80) | |
| `grantee_kind` | VARCHAR(20) | |
| `grantee_id` | UUID | |
| `access` | VARCHAR(20) | |
| `is_active` | BOOLEAN | |

### 4.5 `shr_grant`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `resource_type` | VARCHAR(80) | Generic |
| `resource_id` | UUID | |
| `grantee_kind` | VARCHAR(20) | |
| `grantee_id` | UUID | |
| `access` | VARCHAR(20) | |
| `reason` | VARCHAR(20) | |

**Unique:** `(tenant_id, resource_type, resource_id, grantee_kind, grantee_id, reason)` where not deleted.

### 4.6 `shr_access_eval`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_ref` | UUID | |
| `resource_type` | VARCHAR(80) | |
| `resource_id` | UUID | |
| `allowed` | BOOLEAN | |
| `matched_reason` | VARCHAR(20) NULL | |

---

## 5. RLS summary

All tables FORCE RLS fail-closed.

---

## 6. Seed minimum

Permissions `sharing.*`. No CRM seed data.

---

## 7. ER overview

```text
shr_team 1──* shr_team_member
shr_role_hierarchy
shr_sharing_rule
shr_grant
shr_access_eval
```

---

## 8. Implementation notes

Alembic `f33a0b1c2d3e` / `f33b1c2d3e4f`. Unique grant prevents duplicate concurrent shares.
