# JeslotERP Extensibility Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f29a0b1c2d3e` / `f29b1c2d3e4f`  
**Package:** `platforms.p29_extensibility`  
**PostgreSQL schema:** `extensibility`  
**Companion:** [`EXTENSIBILITY_GUIDE.md`](EXTENSIBILITY_GUIDE.md) · [`EXTENSIBILITY_API.md`](EXTENSIBILITY_API.md)

> Runtime models: `platforms/p29_extensibility/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `extensibility` (never `p29`) |
| Tables | `ext_*` |
| Cross-schema | UUID + string keys only |
| RLS | FORCE on tenant bindings/executions |
| Safety | No source-code / script columns that are executed |

---

## 2. Complete table inventory (**6 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `ext_point` | Extension points |
| 2 | `ext_handler` | Allow-listed handler metadata |
| 3 | `ext_allow_list` | Permitted handler keys |
| 4 | `ext_binding` | Point↔handler + priority + phase |
| 5 | `ext_failure_policy` | Default policy per point |
| 6 | `ext_execution` | Run log |
| 7 | `ext_outbox` | Outbox |
| 8 | `ext_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| Phase | `BEFORE`, `VALIDATE`, `AFTER`, `COMPENSATE` |
| Binding status | `DRAFT`, `ACTIVE`, `INACTIVE` |
| Handler kind | `BUILTIN` |
| Failure policy | `FAIL_CLOSED`, `CONTINUE`, `COMPENSATE` |
| Execution status | `SUCCEEDED`, `FAILED`, `SKIPPED`, `TIMEOUT` |

---

## 4. Detailed tables

### 4.1 `ext_point`

| Column | Type | Notes |
|---|---|---|
| `point_key` | VARCHAR(150) UNIQUE | `bp.partner.before_save` |
| `description` | VARCHAR(400) NULL | |
| `resource_type` | VARCHAR(80) | Generic |
| `status` | VARCHAR(20) | |

### 4.2 `ext_handler`

| Column | Type | Notes |
|---|---|---|
| `handler_key` | VARCHAR(150) UNIQUE | |
| `kind` | VARCHAR(20) | `BUILTIN` only v1 |
| `timeout_ms` | INTEGER | Default 2000 |
| `is_active` | BOOLEAN | |

### 4.3 `ext_allow_list`

| Column | Type | Notes |
|---|---|---|
| `handler_key` | VARCHAR(150) UNIQUE | Must match handler |
| `reason` | VARCHAR(200) | |

### 4.4 `ext_binding`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system |
| `point_id` | UUID | |
| `handler_id` | UUID | |
| `phase` | VARCHAR(20) | |
| `priority` | INTEGER | Lower first |
| `status` | VARCHAR(20) | |
| `timeout_ms` | INTEGER NULL | Override |
| `failure_policy` | VARCHAR(20) | |

**Partial unique:** `(tenant_id, point_id, handler_id, phase)` where not deleted.

### 4.5 `ext_failure_policy`

| Column | Type | Notes |
|---|---|---|
| `point_id` | UUID | |
| `policy` | VARCHAR(20) | Default for point |

### 4.6 `ext_execution`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `binding_id` | UUID NULL | |
| `point_key` | VARCHAR(150) | |
| `handler_key` | VARCHAR(150) | |
| `phase` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |
| `duration_ms` | INTEGER | |
| `error_code` | VARCHAR(80) NULL | |
| `correlation_id` | VARCHAR(80) NULL | |

---

## 5. RLS summary

Tenant bindings and executions: fail-closed FORCE RLS.

---

## 6. Seed minimum

1. Handler `ext.sample.noop` + allow-list  
2. Point `platform.kernel.ping`  
3. Permissions `extensibility.*` …

---

## 7. ER overview

```text
ext_point 1──* ext_binding *──1 ext_handler
                │
                └──* ext_execution
ext_allow_list
```

---

## 8. Implementation notes

Alembic `f29a0b1c2d3e` / `f29b1c2d3e4f`. No executable script column.
