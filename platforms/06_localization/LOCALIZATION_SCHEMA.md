# JeslotERP Localization Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — messages + ICU overlays HTTP persist on AsyncSession. 62 ORM tables + Alembic `a9b0c1d2e3f4` / RLS `b0c1d2e3f4a5`. Not Production.  
**Package:** `platforms.p06_localization`  
**PostgreSQL schema:** `i18n`  
**Companion:** [`LOCALIZATION_GUIDE.md`](LOCALIZATION_GUIDE.md) · [`LOCALIZATION_API.md`](LOCALIZATION_API.md)

> Runtime models: `platforms/p06_localization/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `i18n` (never `p06`) |
| Tables | `i18n_*` |
| Locale IDs | BCP-47 (`hi-IN`) |
| Message keys | Dot namespaces (`bp.partner.gstin.label`) |
| Soft delete | partial uniques on live rows |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant-scoped tables |
| Artifacts | Immutable + checksum |
| ICU | MessageFormat AST or source string + parsed cache |

---

## 2. Complete table inventory (**62 tables**)

### 2.1 Locale / CLDR catalog (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `i18n_language` | ISO language (`hi`, `en`, `ar`) |
| 2 | `i18n_script` | Latn, Deva, Arab, … |
| 3 | `i18n_territory` | IN, US, AE, … |
| 4 | `i18n_locale` | BCP-47 locales |
| 5 | `i18n_calendar` | gregory, indian, … |
| 6 | `i18n_currency` | ISO 4217 catalog (labeling; not FX rates) |
| 7 | `i18n_timezone` | IANA TZ catalog subset |
| 8 | `i18n_locale_alias` | Deprecated → canonical |
| 9 | `i18n_fallback_rule` | Default chains |
| 10 | `i18n_channel` | WEB, MOBILE, EMAIL, SMS, PDF, PRINT |

### 2.2 Message catalog (9)

| # | Table | Purpose |
|---|---|---|
| 11 | `i18n_namespace` | bp, org, meta, notify, common, … |
| 12 | `i18n_message_key` | Stable keys |
| 13 | `i18n_message` | Locale-specific ICU source (system/pack) |
| 14 | `i18n_message_variant` | Channel/feature variants |
| 15 | `i18n_message_argument` | Declared ICU args + types |
| 16 | `i18n_message_tag` | Tagging (screen, module) |
| 17 | `i18n_message_key_tag` | M2M |
| 18 | `i18n_message_context` | Translator description |
| 19 | `i18n_message_screenshot` | media_id refs |

### 2.3 Regional formats (7)

| # | Table | Purpose |
|---|---|---|
| 20 | `i18n_format_profile` | Named profile per locale |
| 21 | `i18n_number_format` | Number patterns |
| 22 | `i18n_currency_format` | Currency patterns |
| 23 | `i18n_date_time_format` | Date/time skeletons |
| 24 | `i18n_address_format` | Address line patterns |
| 25 | `i18n_person_name_format` | Name order |
| 26 | `i18n_phone_format` | Display patterns |

### 2.4 Overrides (4)

| # | Table | Purpose |
|---|---|---|
| 27 | `i18n_tenant_message_override` | Tenant message text |
| 28 | `i18n_company_message_override` | Company message text |
| 29 | `i18n_tenant_format_override` | Tenant format profile |
| 30 | `i18n_user_locale_preference` | User locale/format prefs |

### 2.5 Glossary & TM (6)

| # | Table | Purpose |
|---|---|---|
| 31 | `i18n_glossary` | Glossary sets |
| 32 | `i18n_glossary_term` | Terms |
| 33 | `i18n_glossary_translation` | Term per locale |
| 34 | `i18n_tm_entry` | Translation memory |
| 35 | `i18n_tm_usage` | Leverage stats |
| 36 | `i18n_term_violation` | Lint findings |

### 2.6 TMS workflow (8)

| # | Table | Purpose |
|---|---|---|
| 37 | `i18n_translation_job` | Job header |
| 38 | `i18n_translation_task` | Per-key task |
| 39 | `i18n_translation_assignment` | Assignee |
| 40 | `i18n_translation_comment` | Comments |
| 41 | `i18n_translation_review` | Review decisions |
| 42 | `i18n_translation_proposal` | Candidate text (human/MT) |
| 43 | `i18n_workflow_state_history` | Audit of state changes |
| 44 | `i18n_translator_profile` | Skills/locales |

### 2.7 Packs & publish (7)

| # | Table | Purpose |
|---|---|---|
| 45 | `i18n_package` | Language/country packs |
| 46 | `i18n_package_item` | Pack payloads |
| 47 | `i18n_changeset` | Catalog change batches |
| 48 | `i18n_changeset_item` | Ops |
| 49 | `i18n_approval` | Approvals |
| 50 | `i18n_publish_version` | Bundle versions |
| 51 | `i18n_publish_artifact` | Immutable bundles |

### 2.8 Quality / MT / pseudo (7)

| # | Table | Purpose |
|---|---|---|
| 52 | `i18n_coverage_report` | Coverage scan header |
| 53 | `i18n_coverage_finding` | Missing/stale keys |
| 54 | `i18n_lint_rule` | Lint rules |
| 55 | `i18n_mt_provider` | Provider registry (no secrets) |
| 56 | `i18n_mt_run` | MT batch runs |
| 57 | `i18n_mt_suggestion` | MT outputs |
| 58 | `i18n_pseudo_run` | Pseudo-loc generation |

### 2.9 Plumbing & cache (4)

| # | Table | Purpose |
|---|---|---|
| 59 | `i18n_outbox` | Outbox |
| 60 | `i18n_idempotency_key` | Idempotency |
| 61 | `i18n_resolve_cache` | Optional durable cache |
| 62 | `i18n_catalog_audit` | Catalog audit trail |

**Total: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `i18n_text_direction` | `ltr`, `rtl` |
| `i18n_message_status` | `DRAFT`, `ACTIVE`, `DEPRECATED` |
| `i18n_task_status` | `QUEUED`, `IN_TRANSLATION`, `IN_REVIEW`, `APPROVED`, `REJECTED`, `CANCELLED` |
| `i18n_proposal_source` | `HUMAN`, `TM_EXACT`, `TM_FUZZY`, `MT`, `IMPORT` |
| `i18n_argument_type` | `STRING`, `NUMBER`, `DATE`, `TIME`, `CURRENCY`, `BOOL`, `ENUM` |
| `i18n_coverage_severity` | `INFO`, `WARNING`, `BLOCKER` |
| `i18n_package_status` | `AVAILABLE`, `INSTALLED`, `DISABLED` |
| `i18n_channel_code` | `WEB`, `MOBILE`, `EMAIL`, `SMS`, `WHATSAPP`, `PDF`, `PRINT`, `PUSH` |

---

## 4. Locale catalog (detail)

### 4.1 `i18n_locale`

| Column | Type | Notes |
|---|---|---|
| `locale_code` | VARCHAR(35) UNIQUE | `hi-IN` |
| `language_code` | VARCHAR(15) | |
| `script_code` | VARCHAR(15) NULL | |
| `territory_code` | VARCHAR(5) NULL | |
| `direction` | VARCHAR(3) NOT NULL | ltr/rtl |
| `default_calendar_code` | VARCHAR(30) NULL | |
| `is_product_supported` | BOOLEAN | Shown in UI picker |
| `is_pseudo` | BOOLEAN DEFAULT false | `en-XA` |
| `cldr_version` | VARCHAR(20) NULL | Seed provenance |
| `name_en` | VARCHAR(100) | |
| `native_name` | VARCHAR(100) NULL | |

### 4.2 `i18n_fallback_rule`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system default |
| `requested_locale` | VARCHAR(35) | |
| `chain` | JSONB | `["hi-IN","hi","en-IN","en"]` |
| `is_active` | BOOLEAN | |

---

## 5. Message catalog (detail)

### 5.1 `i18n_namespace`

| Column | Type | Notes |
|---|---|---|
| `namespace_key` | VARCHAR(100) UNIQUE | `bp`, `meta`, `notify.email` |
| `name` | VARCHAR(150) | |
| `is_critical` | BOOLEAN | Coverage gate |
| `owner_team` | VARCHAR(100) NULL | |

### 5.2 `i18n_message_key`

| Column | Type | Notes |
|---|---|---|
| `namespace_id` | UUID | |
| `message_key` | VARCHAR(200) | Full key or relative; unique with namespace |
| `description` | TEXT NULL | Dev context |
| `status` | VARCHAR(20) | |
| `is_icu` | BOOLEAN DEFAULT true | |
| `max_length` | INT NULL | SMS constraints |
| `allows_html` | BOOLEAN DEFAULT false | Strict sanitize if true |
| `deprecated_replacement_key` | VARCHAR(200) NULL | |
| `feature_flag_key` | VARCHAR(100) NULL | |

**Unique:** `(namespace_id, message_key)` active.

### 5.3 `i18n_message`

| Column | Type | Notes |
|---|---|---|
| `message_key_id` | UUID | |
| `locale_code` | VARCHAR(35) | |
| `icu_source` | TEXT NOT NULL | |
| `icu_ast` | JSONB NULL | Parsed cache |
| `checksum` | VARCHAR(64) | |
| `lifecycle` | VARCHAR(20) | DRAFT/PUBLISHED |
| `tenant_id` | UUID NULL | NULL for system |
| `source_locale` | VARCHAR(35) NULL | Usually `en` |
| `revised_at` | TIMESTAMPTZ | |

**Unique:** active `(message_key_id, locale_code, COALESCE(tenant_id,zero), channel_default)`.

### 5.4 `i18n_message_variant`

| Column | Type | Notes |
|---|---|---|
| `message_id` | UUID | Base message |
| `channel_code` | VARCHAR(20) NULL | |
| `feature_flag_key` | VARCHAR(100) NULL | |
| `icu_source` | TEXT | |
| `priority` | INT | |

### 5.5 `i18n_message_argument`

| Column | Type | Notes |
|---|---|---|
| `message_key_id` | UUID | |
| `arg_name` | VARCHAR(50) | |
| `arg_type` | VARCHAR(20) | |
| `example_value` | JSONB NULL | |
| `is_required` | BOOLEAN | |

### 5.6 Screenshots / context

`i18n_message_context`: `notes`, `screen_key`, `url`.  
`i18n_message_screenshot`: `media_id`, `caption`, access via permission.

---

## 6. Formats (detail)

### 6.1 `i18n_format_profile`

| Column | Type | Notes |
|---|---|---|
| `profile_key` | VARCHAR(100) | `hi-IN.default` |
| `locale_code` | VARCHAR(35) | |
| `tenant_id` | UUID NULL | |
| `is_default` | BOOLEAN | |

Child tables hold patterns:

- `i18n_number_format`: `decimal_sep`, `group_sep`, `pattern`, `min_fraction`, `max_fraction`  
- `i18n_currency_format`: `currency_code`, `pattern`, `accounting_pattern`  
- `i18n_date_time_format`: `date_short/medium/long`, `time_short/medium`, `hour_cycle` (`h12`/`h23`), `first_day_of_week`  
- `i18n_address_format`: `lines` JSONB ordered tokens  
- `i18n_person_name_format`: `order_pattern`  
- `i18n_phone_format`: `display_pattern`  

---

## 7. Overrides

### 7.1 `i18n_tenant_message_override` / `company_…`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | |
| `company_id` | UUID NULL | company table |
| `message_key_id` | UUID | |
| `locale_code` | VARCHAR(35) | |
| `icu_source` | TEXT | |
| `status` | VARCHAR(20) | ACTIVE |
| `approved_by` | UUID NULL | |

FORCE RLS on tenant_id.

### 7.2 `i18n_user_locale_preference`

| Column | Type | Notes |
|---|---|---|
| `user_id` | UUID | IAM ref |
| `tenant_id` | UUID | |
| `ui_locale` | VARCHAR(35) NULL | |
| `format_locale` | VARCHAR(35) NULL | May differ |
| `timezone` | VARCHAR(100) NULL | Prefer IAM profile; optional mirror |

---

## 8. Glossary & TM

### 8.1 `i18n_glossary_term`

| Column | Type | Notes |
|---|---|---|
| `glossary_id` | UUID | |
| `term` | VARCHAR(150) | Source locale term |
| `definition` | TEXT NULL | |
| `case_sensitive` | BOOLEAN | |
| `forbidden` | BOOLEAN | Do-not-use |
| `required_translation` | BOOLEAN | Must match glossary |

### 8.2 `i18n_tm_entry`

| Column | Type | Notes |
|---|---|---|
| `source_locale` | VARCHAR(35) | |
| `target_locale` | VARCHAR(35) | |
| `source_hash` | VARCHAR(64) | |
| `source_text` | TEXT | |
| `target_text` | TEXT | |
| `quality` | INT | 0–100 |
| `namespace_id` | UUID NULL | |
| `last_used_at` | TIMESTAMPTZ | |

---

## 9. TMS tables

### 9.1 `i18n_translation_job`

| Column | Type | Notes |
|---|---|---|
| `job_key` | VARCHAR(100) | |
| `source_locale` | VARCHAR(35) | usually `en` |
| `target_locale` | VARCHAR(35) | |
| `namespace_id` | UUID NULL | |
| `status` | VARCHAR(30) | |
| `due_at` | TIMESTAMPTZ NULL | |
| `tenant_id` | UUID NULL | system or tenant |

### 9.2 `i18n_translation_task`

| Column | Type | Notes |
|---|---|---|
| `job_id` | UUID | |
| `message_key_id` | UUID | |
| `status` | VARCHAR(30) | |
| `selected_proposal_id` | UUID NULL | |

### 9.3 `i18n_translation_proposal`

| Column | Type | Notes |
|---|---|---|
| `task_id` | UUID | |
| `source` | VARCHAR(20) | HUMAN/TM/MT/IMPORT |
| `icu_source` | TEXT | |
| `tm_score` | NUMERIC(5,2) NULL | |
| `created_by` | UUID NULL | |

---

## 10. Packs & publish

### 10.1 `i18n_package`

| Column | Type | Notes |
|---|---|---|
| `package_key` | VARCHAR(150) | `lang.hi-IN` |
| `version` | VARCHAR(50) | semver |
| `locale_code` | VARCHAR(35) NULL | |
| `checksum` | VARCHAR(64) | |
| `signature` | TEXT NULL | |
| `status` | VARCHAR(20) | |

### 10.2 `i18n_publish_version` / `artifact`

Artifacts types: `MESSAGES_BUNDLE`, `FORMATS_BUNDLE`, `FULL_LOCALE_PACK`.  
Payload JSONB or compressed bytea reference via media_id.  
`parent_version_number` for rollback lineage.  
Required `checksum`.

---

## 11. Coverage / MT / pseudo

### 11.1 `i18n_coverage_finding`

| Column | Type | Notes |
|---|---|---|
| `report_id` | UUID | |
| `message_key_id` | UUID | |
| `locale_code` | VARCHAR(35) | |
| `finding_type` | VARCHAR(40) | MISSING, STALE, ICU_INVALID, GLOSSARY_VIOLATION, MAX_LENGTH |
| `severity` | VARCHAR(20) | |

### 11.2 `i18n_mt_provider`

| Column | Type | Notes |
|---|---|---|
| `provider_key` | VARCHAR(50) | `google`, `azure`, `openai` |
| `secret_ref_key` | VARCHAR(150) | configuration secret key name |
| `is_active` | BOOLEAN | |

### 11.3 `i18n_pseudo_run`

Generates `en-XA` / brackets expansion for QA; writes draft messages or ephemeral bundle.

---

## 12. Plumbing

- `i18n_outbox`  
- `i18n_idempotency_key`  
- `i18n_resolve_cache` — `(cache_key, etag, payload, expires_at, tenant_id)`  
- `i18n_catalog_audit` — before/after JSON for key/message/override changes  

---

## 13. RLS summary

| Class | Policy |
|---|---|
| System locale/message seeds | Readable authenticated; manage via permission |
| Tenant/company overrides, tenant jobs | FORCE RLS `tenant_id` |
| Screenshots metadata | Tenant RLS + permission |
| Publish artifacts system | Readable; writable publish permission |

---

## 14. Seed minimum

1. Languages: `en`, `hi`, `ar` (+ scripts/territories IN/US/AE)  
2. Locales: `en`, `en-IN`, `en-US`, `hi-IN`, `ar-AE`, `en-XA` (pseudo)  
3. Fallback chains for above  
4. Channels list  
5. Namespaces: `common`, `org`, `bp`, `meta`, `notify`, `auth`, `validation`  
6. Critical English messages for auth + common errors  
7. Format profiles for `en-IN`, `hi-IN`  
8. Permission catalog `i18n.*`  
9. Lint rules: trailing space, empty ICU, glossary forbidden  

---

## 15. ER overview

```text
language/script/territory ──► locale ──► fallback_rule
namespace ──► message_key ──┬── message (per locale)
                            ├── arguments / tags / context / screenshots
                            └── variants

format_profile ── number/currency/date/address/name/phone

glossary ── terms ── translations
tm_entry

job ── tasks ── proposals / reviews / comments
package ── items
changeset ── approval ── publish_version ── artifacts

coverage_report ── findings
mt_run ── suggestions
```

---

## 16. Implementation notes

1. Parse ICU on write; store `icu_ast`; fail fast on invalid.  
2. Runtime resolve should prefer published artifacts over live DRAFT rows.  
3. Keep MT API keys in configuration secrets; only `secret_ref_key` here.  
4. Hindi/Arabic plural rules must be covered by engine tests.  
5. SMS channel enforces `max_length` lint.  
6. Split models: `cldr`, `catalog`, `formats`, `overrides`, `tms`, `packs`, `quality`, `plumbing`.
