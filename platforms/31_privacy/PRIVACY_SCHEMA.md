# JeslotERP Privacy Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f31a0b1c2d3e` / `f31b1c2d3e4f`  
**Package:** `platforms.p31_privacy`  
**PostgreSQL schema:** `privacy`  
**Companion:** [`PRIVACY_GUIDE.md`](PRIVACY_GUIDE.md) · [`PRIVACY_API.md`](PRIVACY_API.md)

> Runtime models: `platforms/p31_privacy/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `privacy` (never `p31`) |
| Tables | `prv_*` |
| Cross-schema | `legal_hold_ref` UUID → p19; no FK |
| RLS | FORCE on all tenant tables |

---

## 2. Complete table inventory (**7 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `prv_subject` | Data subject |
| 2 | `prv_purpose` | Processing purpose |
| 3 | `prv_consent` | Consent grant/withdraw |
| 4 | `prv_policy` | Privacy policy version |
| 5 | `prv_dsr` | DSR request |
| 6 | `prv_operation` | Orchestration step |
| 7 | `prv_legal_hold_ref` | Active hold pointer |
| 8 | `prv_outbox` | Outbox |
| 9 | `prv_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| DSR kind | `ACCESS`, `ERASURE`, `RESTRICT` |
| DSR status | `OPEN`, `IN_PROGRESS`, `COMPLETED`, `BLOCKED`, `REJECTED` |
| Consent status | `GRANTED`, `WITHDRAWN` |
| Op status | `PENDING`, `DONE`, `SKIPPED`, `FAILED` |

---

## 4. Detailed tables

### 4.1 `prv_subject`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `subject_key` | VARCHAR(120) | Unique per tenant |
| `display_ref` | VARCHAR(200) NULL | Non-PII label preferred |

### 4.2 `prv_purpose`

| Column | Type | Notes |
|---|---|---|
| `purpose_key` | VARCHAR(80) UNIQUE | |
| `name` | VARCHAR(120) | |

### 4.3 `prv_consent`

| Column | Type | Notes |
|---|---|---|
| `subject_id` | UUID | |
| `purpose_id` | UUID | |
| `status` | VARCHAR(20) | |
| `withdrawn_at` | TIMESTAMPTZ NULL | |

### 4.4 `prv_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(80) | |
| `version_no` | INTEGER | |
| `body` | TEXT | |
| `is_active` | BOOLEAN | |

### 4.5 `prv_dsr`

| Column | Type | Notes |
|---|---|---|
| `subject_id` | UUID | |
| `kind` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |
| `block_reason` | VARCHAR(80) NULL | `LEGAL_HOLD` |

### 4.6 `prv_operation`

| Column | Type | Notes |
|---|---|---|
| `dsr_id` | UUID | |
| `platform_ref` | VARCHAR(40) | e.g. `p04` |
| `status` | VARCHAR(20) | |
| `detail` | JSONB | |

### 4.7 `prv_legal_hold_ref`

| Column | Type | Notes |
|---|---|---|
| `subject_id` | UUID | |
| `hold_ref` | UUID | p19 legal hold id |
| `is_active` | BOOLEAN | |

---

## 5. RLS summary

All tenant tables FORCE RLS fail-closed.

---

## 6. Seed minimum

Purposes `OPERATIONS`, `SUPPORT`. Permissions `privacy.*`.

---

## 7. ER overview

```text
prv_subject 1──* prv_consent *──1 prv_purpose
     │
     ├──* prv_dsr 1──* prv_operation
     └──* prv_legal_hold_ref
```

---

## 8. Implementation notes

Alembic `f31a0b1c2d3e` / `f31b1c2d3e4f`. Erase adapters do not DELETE foreign schemas in v1.
