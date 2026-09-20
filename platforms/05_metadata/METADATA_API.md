# JeslotERP Metadata Platform — Complete API Specification (Advanced)

**Version:** 2.0  
**Last reviewed:** 2026-09-09  
**Status:** **Live** (routers mounted via `p05_metadata`; contract tests in `platforms/p05_metadata/tests`)  
**Package:** `platforms.p05_metadata`  
**Public base:** `/api/v1/metadata`  
**Internal base:** `/internal/v1/metadata`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`METADATA_GUIDE.md`](METADATA_GUIDE.md) · [`METADATA_SCHEMA.md`](METADATA_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-09 | Dictionary + UI + publish baseline. |
| **2.0** | **2026-09-09** | Effective resolver, FLS, impact, drift, packages, approvals, semantic/CQRS descriptors, variants, cache/ETag. |

---

## 1. Design principles (advanced)

1. **Effective-first** — production clients use `/effective/*` and `/ui-pack`; raw catalog is for authors.  
2. **Layer provenance** — every effective field includes `origin_layer`.  
3. **Immutable publish** — artifacts addressed by version + checksum; rollback publishes prior.  
4. **Safe validate** — AST allow-list; deterministic; no side effects.  
5. **Impact gates** — breaking changes require `metadata.publish` + explicit `allow_breaking=true`.  
6. **FLS redaction** — servers redact/mask by policy; UI hides controls by descriptor.  
7. **Channel/role variants** — resolver picks form/list/action variants.  
8. **ETag / If-None-Match** on ui-pack and effective entity.  
9. **Idempotency** on create/publish/package install.  
10. **Draft isolation** — `lifecycle=draft` requires manage permission.

---

## 2. Common headers & query context

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
If-Match: <version>
If-None-Match: <etag>
```

### Effective context query (optional overrides; never for auth)

| Param | Meaning |
|---|---|
| `channel` | `WEB_DENSE` (default), `MOBILE`, … |
| `include_overlays` | default `true` |
| `include_user_prefs` | default `true` |
| `feature_flags` | comma list (else resolved server-side via p12 when wired) |
| `publish_version` | pin system version |
| `locale` | for future label hydration |
| `lifecycle` | `published` (default) \| `draft` |

Authz still uses JWT tenant/company/roles.

---

## 3. Errors (extended)

```text
METADATA_*_NOT_FOUND
METADATA_KEY_EXISTS / KEY_IMMUTABLE
METADATA_PUBLISH_CONFLICT / BREAKING_CHANGE_BLOCKED
METADATA_APPROVAL_REQUIRED / APPROVAL_REJECTED
METADATA_AST_INVALID / AST_FORBIDDEN_NODE
METADATA_PACKAGE_CHECKSUM_MISMATCH / PACKAGE_SIGNED_REJECTED
METADATA_DRIFT_SCAN_RUNNING
METADATA_FLS_DENIED
METADATA_VERSION_CONFLICT / IDEMPOTENCY_CONFLICT
METADATA_EXPRESSION_TIMEOUT
```

---

## 4. Permissions (extended)

| Code | Use |
|---|---|
| `metadata.catalog.read` | Read published/effective |
| `metadata.catalog.manage` | Mutate system catalog |
| `metadata.extension.read` / `manage` | Tenant extensions/overlays |
| `metadata.pack.install` | Install packs |
| `metadata.publish` | Publish/rollback |
| `metadata.approve` | Approve changesets |
| `metadata.validate` | Validate API |
| `metadata.impact.read` | Impact graph |
| `metadata.drift.read` | Drift |
| `metadata.security.manage` | FLS policies |
| `metadata.audit.read` | Catalog audit |
| `metadata.*` | All |

---

## 5. Effective resolution APIs (primary runtime)

### 5.1 UI pack

```http
GET /api/v1/metadata/ui-pack?entity_key=bp.partner&form_key=bp.partner.edit&list_key=bp.partner.default&channel=WEB_DENSE
```

Permission: `metadata.catalog.read`

Response includes `etag`, `publish_version`, `checksum`, `entity`, `fields[]` (with `origin_layer`, `security`, `semantic_type_code`), `form`, `list_view`, `filters`, `actions`, `inspector`.

Supports `If-None-Match` → `304`.

### 5.2 Effective entity

```http
GET /api/v1/metadata/effective/entities/{entity_key}
```

### 5.3 Effective form / list / actions

```http
GET /api/v1/metadata/effective/forms/{form_key}
GET /api/v1/metadata/effective/list-views/{list_key}
GET /api/v1/metadata/effective/actions?entity_key=bp.partner
```

### 5.4 Resolve debug (authors)

```http
GET /api/v1/metadata/effective/debug?entity_key=bp.partner
```

Permission: manage — returns layer merge trace.

---

## 6. Validate API (advanced)

```http
POST /api/v1/metadata/validate
```

```json
{
  "entity_key": "bp.partner",
  "mode": "CREATE",
  "channel": "WEB_DENSE",
  "payload": { "partner_code": "X", "gstin": "bad", "custom_fields": { "fleet_size": 3 } },
  "evaluate_computed": true,
  "include_warnings": true
}
```

Response:

```json
{
  "valid": false,
  "normalized_payload": {},
  "computed": { "display_label": "X" },
  "errors": [{ "field_key": "gstin", "code": "…", "severity": "ERROR", "ast_rule_key": "…" }],
  "warnings": [],
  "fls_stripped_fields": ["pan_no"]
}
```

Unknown custom keys ⇒ error (default).  
FLS write-denied fields stripped and listed.

---

## 7. Dictionary authoring APIs

Same surface as v1 for:

- `/modules`
- `/entities`, `/entities/{key}/fields`, options, relations
- plus **semantic** fields on create/patch (`semantic_type_code`, `classification_code`, …)

### Deprecate with replacement

```http
POST /api/v1/metadata/entities/{entity_key}/fields/{field_key}/deprecate
{ "replacement_field_key": "gstin_v2", "allow_breaking": false }
```

If dependents exist and `allow_breaking=false` → `409 METADATA_BREAKING_CHANGE_BLOCKED` with impact summary.

---

## 8. Semantic layer APIs

| Method | Path | Permission |
|---|---|---|
| GET/POST | `/business-domains` | read / manage |
| GET/POST | `/semantic-types` | read / manage |
| GET/POST | `/measures` | read / manage |
| GET/POST | `/dimensions` | read / manage |
| GET | `/semantic/entities/{entity_key}` | read — measures+dimensions for entity |

---

## 9. Expressions API

| Method | Path | Permission |
|---|---|---|
| POST | `/expressions` | manage / extension manage |
| POST | `/expressions/validate-ast` | manage — dry parse |
| GET | `/expressions/{expression_key}` | read |
| POST | `/expressions/{expression_key}/evaluate` | manage — test eval with sample context |

```json
{
  "expression_key": "bp.partner.gstin.visible",
  "purpose": "VISIBILITY",
  "ast": {
    "op": "AND",
    "args": [
      { "op": "EQ", "args": [{ "op": "FIELD_REF", "name": "partner_type" }, { "op": "LITERAL", "value": "ORGANIZATION" }] },
      { "op": "FEATURE_ENABLED", "flag": "bp.gst.enabled" }
    ]
  }
}
```

Forbidden nodes → `METADATA_AST_FORBIDDEN_NODE`.

---

## 10. Security (FLS) APIs

| Method | Path | Permission |
|---|---|---|
| GET/PUT | `/fields/{entity_key}/{field_key}/security` | security.manage |
| GET/POST | `/masking-policies` | security.manage |
| GET/PUT | `/entities/{entity_key}/security` | security.manage |
| POST | `/redact` | internal/public validate-style helper |

`POST /redact` body: `{ "entity_key": "…", "rows": [{}] }` → masked rows per caller permissions.

---

## 11. UI authoring APIs (extended)

v1 forms/lists/filters/actions **plus**:

| Method | Path | Purpose |
|---|---|---|
| PUT | `/forms/{form_key}/variants` | Channel/role variants |
| PUT | `/list-views/{list_key}/variants` | |
| PUT | `/actions/{action_key}/variants` | |
| GET/PUT | `/inspectors/{inspector_key}` | Inspector layouts |
| GET/PUT | `/user-preferences/{list_key}` | Current user prefs |

---

## 12. Overlays & extensions

| Method | Path | Permission |
|---|---|---|
| GET/PUT/DELETE | `/overlays` | extension.* |
| CRUD | `/entities/{entity_key}/fields` (custom) | extension.manage |
| GET/PUT | `/extension-values` | extension.* (EAV) |

---

## 13. Packages API

| Method | Path | Permission |
|---|---|---|
| GET | `/packages` | read |
| GET | `/packages/{package_key}` | read |
| POST | `/packages/import` | pack.install — upload JSON/YAML payload |
| POST | `/packages/{package_key}/install` | pack.install |
| POST | `/packages/{package_key}/uninstall` | pack.install |
| GET | `/packages/{package_key}/export` | read/manage |

Install is transactional: verify checksum → apply items as changeset → optional auto-publish tenant scope.

---

## 14. Changesets, approvals, publish

### Changesets

| Method | Path |
|---|---|
| GET/POST | `/changesets` |
| POST | `/changesets/{id}/items` |
| POST | `/changesets/{id}/submit` | creates approval PENDING |
| POST | `/changesets/{id}/apply` | requires APPROVED (or bypass permission) |
| POST | `/changesets/{id}/cancel` |

### Approvals

| Method | Path | Permission |
|---|---|---|
| GET | `/approvals` | approve/publish |
| POST | `/approvals/{id}/approve` | `metadata.approve` |
| POST | `/approvals/{id}/reject` | `metadata.approve` |

### Publish versions

| Method | Path | Permission |
|---|---|---|
| GET | `/publish-versions` | read |
| GET | `/publish-versions/latest` | read |
| GET | `/publish-versions/{n}/artifact` | read |
| POST | `/publish-versions` | publish |
| POST | `/publish-versions/{n}/rollback` | publish — creates new version pointing prior artifact |

Publish body:

```json
{
  "scope": "SYSTEM",
  "changeset_id": "UUID",
  "version_label": "2026.09.09.2",
  "allow_breaking": false,
  "notes": "Add bp.partner semantic GSTIN"
}
```

---

## 15. Impact & dependency APIs

```http
GET /api/v1/metadata/impact?target_type=FIELD&target_key=bp.partner.gstin
GET /api/v1/metadata/dependencies?entity_key=bp.partner
POST /api/v1/metadata/dependencies/rebuild
```

Permissions: `metadata.impact.read` (rebuild: manage).

Response:

```json
{
  "target": { "type": "FIELD", "key": "bp.partner.gstin" },
  "breaking": true,
  "dependents": [
    { "type": "FORM_FIELD", "key": "bp.partner.edit#gstin", "is_breaking": true },
    { "type": "VALIDATION", "key": "bp.partner.gstin.regex", "is_breaking": true }
  ]
}
```

---

## 16. Drift APIs

| Method | Path | Permission |
|---|---|---|
| POST | `/drift/scans` | drift.read (+ manage to run) |
| GET | `/drift/scans` | drift.read |
| GET | `/drift/scans/{id}` | drift.read |
| GET | `/drift/scans/{id}/findings` | drift.read |

Scan compares `NATIVE_COLUMN` metadata to live DB.

---

## 17. Interop descriptors

| Method | Path |
|---|---|
| GET/POST | `/commands` |
| GET/POST | `/queries` |
| GET/POST | `/events` |
| GET | `/entities/{entity_key}/contracts` | bundled command/query/event list |

Used by docs portal, gateway codegen, AI tools.

---

## 18. Catalog audit

```http
GET /api/v1/metadata/audit?target_key=bp.partner&page=1
```

Permission: `metadata.audit.read`

---

## 19. Lookups

```http
GET /lookups/data-types
GET /lookups/semantic-types
GET /lookups/ui-controls
GET /lookups/validation-types
GET /lookups/channels
GET /lookups/classifications
GET /lookups/storage-strategies
GET /lookups/relation-kinds
GET /lookups/mask-strategies
```

---

## 20. Internal mesh API

Base: `/internal/v1/metadata` · `X-Internal-Token`

| Method | Path | Purpose |
|---|---|---|
| GET | `/ui-pack` | Effective pack for service |
| GET | `/effective/entities/{entity_key}` | |
| POST | `/validate` | |
| POST | `/redact` | Batch mask |
| GET | `/publish-versions/latest` | |
| GET | `/semantic/entities/{entity_key}` | Measures/dimensions |
| GET | `/contracts/{entity_key}` | CQRS descriptors |
| POST | `/drift/scans` | System job trigger |
| GET | `/health` | |

Internal callers pass explicit context: `tenant_id`, `company_id`, `role_codes`, `channel`.

---

## 21. Caching headers

```http
ETag: "sha256-…"
Cache-Control: private, max-age=60
```

`If-None-Match` on ui-pack/effective GETs.

Invalidate via outbox consumers on publish/overlay/pack/security change.

---

## 22. Example: advanced partner UI flow

1. `GET /ui-pack?entity_key=bp.partner&channel=WEB_DENSE`  
2. Render form using effective fields; hide FLS-denied  
3. On submit `POST /validate` with payload  
4. On valid → `POST /api/v1/bp/partners`  
5. Admin adds custom field → extension.manage → tenant sees field on next ui-pack (new etag)  
6. Platform deprecates field → `GET /impact` → publish with `allow_breaking` if needed  

---

## 23. Example: industry pack install

1. `POST /packages/import` with signed payload `india.gst.bp@1.2.0`  
2. Approval (if tenant policy requires)  
3. `POST /packages/india.gst.bp/install`  
4. Outbox `metadata.package.installed`  
5. Effective resolver layer `INDUSTRY_PACK` adds GST fields/forms  

---

## 24. Implementation checklist (v2)

- [x] Effective resolver with provenance + tests for layer precedence  
- [x] AST parse/eval/forbid suite  
- [ ] FLS redact helper + BP consumer spike  
- [x] Publish immutability + rollback  
- [x] Approval gate  
- [x] Impact graph rebuild  
- [x] Drift scanner against Postgres  
- [x] Package install transaction  
- [x] ETag ui-pack  
- [x] Internal mesh parity  
- [x] Permission seed + FORCE RLS  
- [x] Contract tests for breaking change block  

---

## 25. Boundary reminders

| Need | API |
|---|---|
| Setting value | p03 `/configuration` |
| Partner row | p04 `/bp` |
| How to render/validate partner | p05 `/metadata/ui-pack` + `/validate` |
| Labels | p06 via `label_key` |
| Flag enabled? | p12 (resolver may call) |
