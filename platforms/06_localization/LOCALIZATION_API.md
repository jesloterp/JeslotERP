# JeslotERP Localization Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `/messages` and `/overrides/messages` Postgres-first when session is AsyncSession. Empty list is `[]`. Not Production.  
**Package:** `platforms.p06_localization`  
**PostgreSQL schema:** `i18n`  
**Public base:** `/api/v1/i18n`  
**Internal base:** `/internal/v1/i18n`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`LOCALIZATION_GUIDE.md`](LOCALIZATION_GUIDE.md) · [`LOCALIZATION_SCHEMA.md`](LOCALIZATION_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Effective resolve + ETag, ICU format, formats, TMS, glossary/TM, packs/publish, coverage, MT, pseudo-loc, XLIFF, overrides. |
| 1.1 | 2026-09-12 | TASK-SOR-008: messages + ICU overlays durable; empty catalog is `[]`. |

---

## 1. Design principles (advanced)

1. **Effective-first** — production clients use `/effective/*` and `/bundles/*`; raw catalog is for authors/translators.  
2. **Layer provenance** — every resolved message includes `origin_layer` when `debug=true`.  
3. **ICU MessageFormat** — format endpoint is first-class; never concatenate on clients for plurals.  
4. **Locale fallback** — deterministic chain; never invent opaque language codes at runtime.  
5. **Immutable publish** — runtime prefers published artifacts; drafts stay in TMS.  
6. **Formats ≠ messages** — regional formats resolve on their own path/layer stack.  
7. **Channel / feature variants** — SMS vs WEB vs PDF pick variants without key forking.  
8. **ETag / If-None-Match** on bundles and effective packs.  
9. **Idempotency** on create, pack install, publish, MT run, import.  
10. **MT is assist-only** — proposals never become production without review policy.  
11. **Glossary gates** — submit/approve can fail on forbidden/required term violations.  
12. **Coverage gates** — publish may block on `BLOCKER` findings for critical namespaces.  
13. **RTL as data** — `direction` always returned with locale metadata.  
14. **Secrets out of band** — MT credentials via configuration secrets; i18n stores `secret_ref_key` only.

---

## 2. Common headers & query context

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept-Language: hi-IN,hi;q=0.9,en;q=0.8
X-Request-ID: <uuid>
Idempotency-Key: <key>
If-Match: <version>
If-None-Match: <etag>
X-Tenant-Id: <uuid>          # when platform requires explicit tenant header
X-Company-Id: <uuid>         # optional company override layer
```

### Effective context query (optional; never for authz)

| Param | Meaning |
|---|---|
| `locale` | Requested BCP-47 (`hi-IN`). Default: user preference → company → tenant → `en` |
| `format_locale` | Optional; may differ from UI locale |
| `channel` | `WEB` (default), `MOBILE`, `EMAIL`, `SMS`, `WHATSAPP`, `PDF`, `PRINT`, `PUSH` |
| `namespace` | Restrict resolve |
| `keys` | Comma-separated key list |
| `prefix` | Key prefix filter (`bp.partner.`) |
| `include_formats` | default `true` |
| `include_overlays` | tenant/company overrides; default `true` |
| `include_user_prefs` | default `true` |
| `feature_flags` | comma list (else resolved via p12 when wired) |
| `publish_version` | Pin bundle version |
| `lifecycle` | `published` (default) \| `draft` (requires manage/translate) |
| `debug` | Include `fallback_trace` + `origin_layer` |

Authz always uses JWT tenant/roles; `Accept-Language` is a **hint**, not authorization.

---

## 3. Envelope

All public endpoints return `StandardResponse`:

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "request_id": "…",
    "etag": "W/\"i18n-bundle-…\"",
    "bundle_version": 42,
    "checksum": "sha256:…"
  }
}
```

---

## 4. Errors

```text
I18N_LOCALE_NOT_FOUND / LOCALE_UNSUPPORTED
I18N_NAMESPACE_NOT_FOUND / KEY_NOT_FOUND / KEY_EXISTS / KEY_IMMUTABLE
I18N_ICU_INVALID / ICU_FORBIDDEN_NODE / ARGUMENT_MISMATCH
I18N_FALLBACK_EXHAUSTED
I18N_FORMAT_PROFILE_NOT_FOUND
I18N_OVERRIDE_CONFLICT / OVERRIDE_DENIED
I18N_JOB_NOT_FOUND / TASK_NOT_FOUND / INVALID_TASK_STATE
I18N_GLOSSARY_VIOLATION / TERM_FORBIDDEN
I18N_TM_MISS
I18N_PACKAGE_NOT_FOUND / CHECKSUM_MISMATCH / SIGNATURE_REJECTED / ALREADY_INSTALLED
I18N_PUBLISH_CONFLICT / COVERAGE_BLOCKED / APPROVAL_REQUIRED
I18N_MT_PROVIDER_UNAVAILABLE / MT_DENIED
I18N_IMPORT_INVALID / XLIFF_SCHEMA_INVALID
I18N_VERSION_CONFLICT / IDEMPOTENCY_CONFLICT
I18N_PSEUDO_DENIED
I18N_FLS_DENIED
```

HTTP mapping: `404` not found · `409` conflict/state · `422` ICU/glossary/validation · `403` permission · `412` If-Match · `304` If-None-Match.

---

## 5. Permissions

| Code | Use |
|---|---|
| `i18n.catalog.read` | Read published catalog / resolve |
| `i18n.catalog.manage` | Manage namespaces, keys, system messages |
| `i18n.translate` | Submit proposals / work tasks |
| `i18n.review` | Approve / reject translations |
| `i18n.publish` | Publish / rollback bundles |
| `i18n.pack.install` | Install / uninstall language packs |
| `i18n.override.manage` | Tenant / company message & format overrides |
| `i18n.glossary.manage` | Glossary CRUD |
| `i18n.tm.manage` | TM admin / purge |
| `i18n.mt.run` | Run machine translation batches |
| `i18n.coverage.read` | Coverage reports |
| `i18n.pseudo.run` | Pseudo-localization |
| `i18n.import_export` | XLIFF/JSON/CSV import-export |
| `i18n.audit.read` | Catalog / TMS audit |
| `i18n.*` | All |

---

## 6. Effective resolution APIs (primary runtime)

### 6.1 Resolve messages + formats pack

```http
GET /api/v1/i18n/effective/pack
```

**Query:** context params from §2.

**Response `data`:**

```json
{
  "locale": "hi-IN",
  "resolved_locale": "hi-IN",
  "direction": "ltr",
  "fallback_chain": ["hi-IN", "hi", "en-IN", "en"],
  "bundle_version": 42,
  "checksum": "sha256:…",
  "messages": {
    "bp.partner.gstin.label": "जीएसटीआईएन",
    "bp.partner.count": "{count, plural, =0 {कोई पार्टनर नहीं} one {# पार्टनर} other {# पार्टनर}}"
  },
  "formats": {
    "number": { "decimal_sep": ".", "group_sep": ",", "pattern": "#,##,##0.###" },
    "currency": { "code": "INR", "pattern": "¤#,##,##0.00" },
    "date": { "short": "dd/MM/yy", "medium": "dd MMM yyyy", "hour_cycle": "h12", "first_day_of_week": 1 },
    "address": { "lines": ["{name}", "{line1}", "{line2}", "{city}, {state} {postal}", "{country}"] },
    "person_name": { "order_pattern": "{given} {family}" }
  },
  "origins": {
    "bp.partner.gstin.label": { "layer": "LANGUAGE_PACK", "pack": "lang.hi-IN@2.1.0" }
  }
}
```

`origins` only when `debug=true`. Supports `If-None-Match` → `304`.

### 6.2 Resolve specific keys

```http
POST /api/v1/i18n/effective/messages
```

```json
{
  "locale": "hi-IN",
  "channel": "WEB",
  "keys": ["bp.partner.gstin.label", "common.save", "validation.required"],
  "debug": false
}
```

### 6.3 Format (ICU evaluate)

```http
POST /api/v1/i18n/effective/format
```

```json
{
  "locale": "hi-IN",
  "key": "bp.partner.count",
  "args": { "count": 3 },
  "channel": "WEB"
}
```

**Response:**

```json
{
  "key": "bp.partner.count",
  "locale": "hi-IN",
  "text": "3 पार्टनर",
  "direction": "ltr",
  "origin_layer": "LANGUAGE_PACK"
}
```

Server-side formatting is preferred for PDF/email; browsers may also evaluate published ICU with a vetted client engine.

### 6.4 Formats only

```http
GET /api/v1/i18n/effective/formats?locale=hi-IN&company_id=…
```

### 6.5 Locale metadata for shell

```http
GET /api/v1/i18n/effective/locale
```

Returns picker-safe locales for tenant + `direction`, `native_name`, `is_rtl`, default formats summary.

### 6.6 Internal hydrate (metadata / notify)

```http
POST /internal/v1/i18n/hydrate-labels
```

```json
{
  "locale": "hi-IN",
  "tenant_id": "…",
  "label_keys": ["bp.partner.gstin", "meta.entity.Partner.name"]
}
```

Used by p05 / p15 gateways — never called from browser.

---

## 7. Locale & CLDR catalog

```http
GET    /api/v1/i18n/locales
GET    /api/v1/i18n/locales/{locale_code}
GET    /api/v1/i18n/languages
GET    /api/v1/i18n/territories
GET    /api/v1/i18n/fallback-rules?locale=hi-IN
PUT    /api/v1/i18n/fallback-rules                 # i18n.catalog.manage (tenant-scoped override)
GET    /api/v1/i18n/channels
```

`GET /locales` supports `product_supported_only=true` (default for UI pickers).

---

## 8. Namespaces, keys, messages (authoring)

### 8.1 Namespaces

```http
GET    /api/v1/i18n/namespaces
POST   /api/v1/i18n/namespaces                    # catalog.manage
GET    /api/v1/i18n/namespaces/{namespace_key}
PATCH  /api/v1/i18n/namespaces/{namespace_key}
```

### 8.2 Message keys

```http
GET    /api/v1/i18n/keys?namespace=bp&q=gstin&status=ACTIVE
POST   /api/v1/i18n/keys
GET    /api/v1/i18n/keys/{message_key}
PATCH  /api/v1/i18n/keys/{message_key}
POST   /api/v1/i18n/keys/{message_key}/deprecate  # body: { "replacement_key": "…" }
```

**Create body:**

```json
{
  "namespace_key": "bp",
  "message_key": "bp.partner.gstin.label",
  "description": "GSTIN field label on partner form",
  "is_icu": false,
  "max_length": null,
  "arguments": []
}
```

After first publish, `message_key` is immutable; use deprecate.

### 8.3 Messages (per locale)

```http
GET    /api/v1/i18n/messages?key=bp.partner.gstin.label&locale=hi-IN
PUT    /api/v1/i18n/messages                      # upsert draft system/tenant authoring
POST   /api/v1/i18n/messages/validate-icu
GET    /api/v1/i18n/messages/{message_id}/variants
PUT    /api/v1/i18n/messages/{message_id}/variants
```

**Validate ICU:**

```json
{
  "icu_source": "{count, plural, one {# item} other {# items}}",
  "arguments": [{ "arg_name": "count", "arg_type": "NUMBER" }]
}
```

Returns AST preview + diagnostics.

### 8.4 Context / screenshots

```http
PUT    /api/v1/i18n/keys/{message_key}/context
POST   /api/v1/i18n/keys/{message_key}/screenshots   # media_id from p08
DELETE /api/v1/i18n/screenshots/{screenshot_id}
```

---

## 9. Tenant / company overrides

```http
GET    /api/v1/i18n/overrides/messages?locale=hi-IN
PUT    /api/v1/i18n/overrides/messages               # i18n.override.manage
DELETE /api/v1/i18n/overrides/messages/{id}
GET    /api/v1/i18n/overrides/formats
PUT    /api/v1/i18n/overrides/formats
GET    /api/v1/i18n/preferences/me
PUT    /api/v1/i18n/preferences/me                   # ui_locale / format_locale
```

Company overrides require `company_id` in body + company membership.

---

## 10. Translation management (TMS)

### 10.1 Jobs

```http
GET    /api/v1/i18n/jobs
POST   /api/v1/i18n/jobs
GET    /api/v1/i18n/jobs/{job_id}
POST   /api/v1/i18n/jobs/{job_id}/cancel
POST   /api/v1/i18n/jobs/{job_id}/generate-tasks     # expand missing keys
```

**Create job:**

```json
{
  "job_key": "hi-IN-bp-2026Q3",
  "source_locale": "en",
  "target_locale": "hi-IN",
  "namespace_key": "bp",
  "due_at": "2026-09-30T00:00:00Z",
  "include_stale": true
}
```

### 10.2 Tasks & proposals

```http
GET    /api/v1/i18n/jobs/{job_id}/tasks?status=IN_TRANSLATION
GET    /api/v1/i18n/tasks/{task_id}
POST   /api/v1/i18n/tasks/{task_id}/assign
POST   /api/v1/i18n/tasks/{task_id}/proposals
POST   /api/v1/i18n/tasks/{task_id}/submit            # select proposal → IN_REVIEW
POST   /api/v1/i18n/tasks/{task_id}/reviews            # approve | reject
GET    /api/v1/i18n/tasks/{task_id}/comments
POST   /api/v1/i18n/tasks/{task_id}/comments
GET    /api/v1/i18n/tasks/{task_id}/tm-suggestions
GET    /api/v1/i18n/tasks/{task_id}/glossary-hits
```

**Proposal:**

```json
{
  "icu_source": "जीएसटीआईएन",
  "source": "HUMAN"
}
```

Submit/approve runs glossary lint; `422 I18N_GLOSSARY_VIOLATION` on hard fail.

### 10.3 Translator profiles

```http
GET    /api/v1/i18n/translators
PUT    /api/v1/i18n/translators/me
```

---

## 11. Glossary & translation memory

```http
GET    /api/v1/i18n/glossaries
POST   /api/v1/i18n/glossaries
GET    /api/v1/i18n/glossaries/{id}/terms
POST   /api/v1/i18n/glossaries/{id}/terms
PUT    /api/v1/i18n/terms/{term_id}
PUT    /api/v1/i18n/terms/{term_id}/translations/{locale}
POST   /api/v1/i18n/glossary/lint                     # body: { locale, icu_source, namespace? }

GET    /api/v1/i18n/tm/search?source_locale=en&target_locale=hi-IN&q=…
POST   /api/v1/i18n/tm/entries                        # tm.manage
DELETE /api/v1/i18n/tm/entries/{id}
```

---

## 12. Packs & publish governance

### 12.1 Packages

```http
GET    /api/v1/i18n/packages
GET    /api/v1/i18n/packages/{package_key}
POST   /api/v1/i18n/packages/{package_key}/install    # Idempotency-Key required
POST   /api/v1/i18n/packages/{package_key}/uninstall
```

**Install body:**

```json
{
  "version": "2.1.0",
  "checksum": "sha256:…",
  "allow_unsigned": false
}
```

### 12.2 Changesets & approvals

```http
GET    /api/v1/i18n/changesets
POST   /api/v1/i18n/changesets
GET    /api/v1/i18n/changesets/{id}
POST   /api/v1/i18n/changesets/{id}/submit
POST   /api/v1/i18n/changesets/{id}/approvals         # approve | reject
```

### 12.3 Publish / rollback

```http
POST   /api/v1/i18n/publish
POST   /api/v1/i18n/publish/rollback
GET    /api/v1/i18n/publish/versions
GET    /api/v1/i18n/publish/versions/{version}/artifacts
GET    /api/v1/i18n/bundles/{version}                   # download immutable artifact (ETag)
```

**Publish body:**

```json
{
  "changeset_id": "…",
  "locales": ["hi-IN"],
  "namespaces": ["bp", "common"],
  "require_coverage_clean": true,
  "notes": "Q3 Hindi BP labels"
}
```

Fails with `I18N_COVERAGE_BLOCKED` if critical namespace has `BLOCKER` findings and gate enabled.

---

## 13. Coverage, lint, pseudo-loc

```http
POST   /api/v1/i18n/coverage/scans
GET    /api/v1/i18n/coverage/reports
GET    /api/v1/i18n/coverage/reports/{id}/findings?severity=BLOCKER
GET    /api/v1/i18n/lint-rules
PUT    /api/v1/i18n/lint-rules/{rule_key}              # catalog.manage
POST   /api/v1/i18n/pseudo/runs                         # i18n.pseudo.run
GET    /api/v1/i18n/pseudo/runs/{id}
```

**Coverage scan:**

```json
{
  "locales": ["hi-IN", "ar-AE"],
  "namespaces": ["bp", "auth", "common"],
  "treat_stale_as_blocker": true
}
```

**Pseudo run:** generates `en-XA` / expansion brackets for UI length QA; never auto-publishes.

---

## 14. Machine translation

```http
GET    /api/v1/i18n/mt/providers
POST   /api/v1/i18n/mt/runs                             # i18n.mt.run
GET    /api/v1/i18n/mt/runs/{id}
GET    /api/v1/i18n/mt/runs/{id}/suggestions
POST   /api/v1/i18n/mt/suggestions/{id}/accept-to-task  # creates HUMAN-reviewable proposal
```

**MT run:**

```json
{
  "provider_key": "azure",
  "source_locale": "en",
  "target_locale": "hi-IN",
  "job_id": "…",
  "message_key_ids": []
}
```

Providers resolve credentials via `secret_ref_key` → p03 configuration secrets.  
There is **no** public “publish MT directly” endpoint.

---

## 15. Import / export (XLIFF / JSON / CSV)

```http
POST   /api/v1/i18n/export
POST   /api/v1/i18n/import
GET    /api/v1/i18n/import/{import_id}
```

**Export:**

```json
{
  "format": "XLIFF_2_1",
  "source_locale": "en",
  "target_locale": "hi-IN",
  "namespace_key": "bp",
  "lifecycle": "published"
}
```

Returns artifact download URL / media_id.  
Import creates draft messages or TMS proposals per policy; never silent publish.

---

## 16. Audit

```http
GET /api/v1/i18n/audit?entity_type=MESSAGE_KEY&entity_id=…&from=…&to=…
```

Requires `i18n.audit.read`.

---

## 17. Internal platform APIs

| Endpoint | Consumer |
|---|---|
| `POST /internal/v1/i18n/hydrate-labels` | p05 metadata UI packs |
| `POST /internal/v1/i18n/hydrate-template` | p15 notification |
| `GET  /internal/v1/i18n/effective/formats` | PDF / report engines |
| `POST /internal/v1/i18n/cache/invalidate` | After publish (ops) |
| `GET  /internal/v1/i18n/health` | Readiness (DB + last publish) |

Internal auth: `X-Internal-Token` + service identity; still pass `tenant_id` explicitly.

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Effective pack / bundles | ETag + CDN/app cache; invalidate on publish/override |
| Resolve cache table | Optional durable; key = hash(tenant, company, locale, channel, version, keyset) |
| Catalog writes | `If-Match` on versioned entities |
| Pack install / publish / import / MT | `Idempotency-Key` mandatory |

---

## 19. Example client flows

### 19.1 ERP shell boot

1. `GET /effective/locale` → populate language picker + RTL shell class  
2. `GET /effective/pack?locale=hi-IN&channel=WEB` with `If-None-Match`  
3. Cache messages; hydrate metadata labels via keys only  

### 19.2 Translator day

1. Open job → list tasks  
2. Fetch TM suggestions + glossary hits  
3. Submit proposal → review approve  
4. Coverage scan → publish changeset  

### 19.3 Tenant rebrand labels

1. `PUT /overrides/messages` for selected keys  
2. Bundle ETag changes; clients refresh  

### 19.4 Bilty PDF

1. Internal hydrate template keys for `hi-IN`  
2. Format ICU with shipment args  
3. Apply `effective/formats` for currency/date lines  

---

## 20. Event hooks (outbox → consumers)

Clients do not poll forever; platforms may subscribe:

| Event | Typical consumer action |
|---|---|
| `i18n.bundle.published` | Drop resolve caches |
| `i18n.pack.installed` | Refresh product_supported locales |
| `i18n.override.changed` | Tenant CDN purge |
| `i18n.coverage.failed` | Block release pipeline |

---

## 21. Compatibility notes

- Public path prefix is `/api/v1/i18n` (schema name), not `/localization`.  
- Prefer key-based resolve over shipping entire English dictionaries in SPA.  
- `Accept-Language` never overrides explicit `locale` query when both present (`locale` wins).  
- Draft lifecycle responses are redacted unless caller has translate/manage.

---

## 22. Related documents

- Guide: [`LOCALIZATION_GUIDE.md`](LOCALIZATION_GUIDE.md)  
- Schema: [`LOCALIZATION_SCHEMA.md`](LOCALIZATION_SCHEMA.md)  
- Metadata: [`../05_metadata/METADATA_API.md`](../05_metadata/METADATA_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
