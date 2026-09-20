# JeslotERP Localization Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — HTTP messages + ICU overlays Postgres-first when session is `AsyncSession`. Empty catalog is `[]`. Not Production.  
**Package:** `platforms.p06_localization`  
**PostgreSQL schema:** `i18n`  
**Depends on:** `p01_identity`, `p03_configuration`  
**Integrates with:** `p02_organization` (tenant/company context), `p05_metadata` (`label_key`), `p08_file_media` (screenshots), `p12_feature`, `p15_notification` (templates), `p18_search`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`LOCALIZATION_SCHEMA.md`](LOCALIZATION_SCHEMA.md) · [`LOCALIZATION_API.md`](LOCALIZATION_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise L10n/i18n control plane: CLDR-class locales, ICU messages, layered resolve, TM/glossary, TMS workflow, regional formats, packs, publish, coverage, MT hooks, RTL, pseudo-loc. |
| 1.1 | 2026-09-12 | TASK-SOR-008: durable messages + ICU overlays; empty list is `[]`; RLS on `require_i18n_access`. |

---

## 1. Purpose (enterprise)

`p06_localization` is JeslotERP’s **globalization & localization control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **Microsoft Dynamics 365** — language packs, regional settings, user language, company address formats  
- **SAP** — language keys (SPRAS), OTR / SE63 translation, CLDR-based formats, locale-dependent UI  
- **Salesforce** — Translation Workbench, named credentials for MT, locale formats, custom label overrides  
- **ICU / Unicode CLDR** — locales, plurals, currencies, calendars, address patterns  

It is **not** a JSON file of English strings. It is the system that makes the ERP correctly:

1. Serve **UI / API / email / WhatsApp / PDF** copy in the user’s language  
2. Apply **regional formats** (number, currency, date, time, calendar, first day of week)  
3. Resolve **ICU MessageFormat** plurals, selects, gender, and rich arguments  
4. Govern translation with **glossary, TM, review, approve, publish**  
5. Support **tenant/company overrides** without forking product English  
6. Ship **language / country packs** (e.g. `hi-IN`, `ar-AE`, India GST labels)  
7. Measure **coverage & freshness**; block release on critical missing keys  
8. Support **RTL**, pseudo-localization, and translator context (screenshots)  
9. Integrate **machine translation** as assist — never silent prod publish without review policy  
10. Power **metadata `label_key`**, notification templates, and report labels from one catalog  

### Owns

| Domain | Examples |
|---|---|
| Locale catalog | BCP-47 locales, languages, scripts, territories, calendars |
| Message catalog | namespaces, keys, ICU messages, variants |
| Regional formats | number/currency/date/time/address/name/phone patterns |
| Translation ops | jobs, assignments, review, TM, glossary |
| Packs & publish | language packs, immutable bundles, rollback |
| Runtime resolve | layered effective messages + formats + ETag |
| Quality | coverage, pseudo-loc, lint rules, forbidden terms |
| Interop | XLIFF/JSON/CSV import-export, MT provider adapters |

### Does **not** own

| Concern | Owner |
|---|---|
| Auth / users | `p01_identity` (stores user `preferred_language` only) |
| Org geo masters as business data | `p02_organization` (i18n may label them) |
| Setting **values** | `p03_configuration` (may store default locale keys) |
| Entity/field dictionary | `p05_metadata` (stores `label_key`; i18n stores text) |
| Number series | `p07_number_series` |
| Actual email/SMS send | `p15_notification` (consumes localized templates) |
| Font binary hosting | `p08_file_media` / CDN |

### Critical split: Metadata vs Localization

| | **Metadata (p05)** | **Localization (p06)** |
|---|---|---|
| Stores | `label_key = "bp.partner.gstin"` | `hi-IN` text for that key |
| Question | What field exists? | How is it worded here? |
| Change cadence | Schema/UI evolution | Linguistic / market packs |

---

## 2. Architectural position

```text
                    ┌──────────────────────────────────────────┐
                    │     EFFECTIVE L10N RESOLVER               │
                    │  SYSTEM → LANG_PACK → TENANT → COMPANY → │
                    │  USER → CHANNEL → FEATURE_FLAG            │
                    │  + locale fallback chain (hi-IN→hi→en)    │
                    └────────────────────┬─────────────────────┘
                                         │
     ┌────────────┬──────────────┬───────┼────────┬─────────────┐
     ▼            ▼              ▼       ▼        ▼             ▼
  Web/ERP      Mobile        PDF/Print  Notify   Metadata    Reports
  UI packs     apps          docs       templates label_key  headers
```

---

## 3. Advanced design principles

1. **BCP-47 first** — `hi-IN`, `en-US`, `ar-AE`; never invent `HINDI` codes for runtime.  
2. **ICU MessageFormat** for all user-facing dynamic strings (plurals/select).  
3. **Layered effective resolve** with provenance on every hit.  
4. **Fallback chain** deterministic and tenant-overridable.  
5. **Publish immutability** — runtime reads published bundles; drafts for translators.  
6. **Key stability** — `message_key` immutable after publish; deprecate + replace.  
7. **Namespace hygiene** — `bp.*`, `org.*`, `meta.*`, `notify.*`, `common.*`.  
8. **Glossary enforcement** — forbidden/required terms lint on submit.  
9. **TM leverage** — suggest prior translations by fuzzy match.  
10. **Regional formats ≠ translations** — formats are first-class resources.  
11. **RTL as locale property** — layout hints for frontend shells.  
12. **Pseudo-localization** for QA (`en-XA` / `[!!! … !!!]`).  
13. **No cross-schema FKs** — UUID refs only.  
14. **RLS fail-closed** on tenant overrides.  
15. **CQRS HTTP** — thin routers; resolver/TMS services in application layer.  
16. **Secrets** — MT API keys via p03/configuration secrets, not i18n tables.  
17. **PII** — translator screenshots may contain PII; access-controlled.  
18. **Idempotent pack install** + checksum.  

---

## 4. Effective resolution model

### 4.1 Message resolve order (later wins for overrides; packs add)

| Priority | Layer | Source |
|---:|---|---|
| 10 | `SYSTEM` | Product English + seeded locales |
| 20 | `LANGUAGE_PACK` | Installed pack (`hi-IN@2.1.0`) |
| 30 | `TENANT` | Tenant overrides |
| 40 | `COMPANY` | Company overrides |
| 50 | `USER` | Rare personal gloss (usually formats only) |
| 60 | `CHANNEL` | `WEB` / `MOBILE` / `EMAIL` / `SMS` / `PDF` variants |
| 70 | `FEATURE_FLAG` | Gated copy |

### 4.2 Locale fallback

Example chain for requested `hi-IN`:

```text
hi-IN → hi → en-IN → en → (key itself as last resort if policy allows)
```

Missing critical namespace keys in production policy → error metric + optional fail-soft English.

### 4.3 Resolve inputs

```text
locale, tenant_id, company_id?, user_id?,
channel, namespace?, keys[] | prefix,
publish_version?, feature_flags[]
```

### 4.4 Resolve outputs

- `messages` map key→resolved ICU string  
- `formats` (number/currency/date/…)  
- `direction` (`ltr`/`rtl`)  
- `fallback_trace` (debug)  
- `etag` / `bundle_version` / `checksum`  

---

## 5. ICU MessageFormat (mandatory)

Supported (v1 engine):

```text
plural, select, selectordinal,
simple arguments {name},
number/date/time format args (skeleton or style),
nesting within allow-listed depth
```

Denied: arbitrary code, network, SQL, unbounded recursion.

Example key `bp.partner.count`:

```text
{count, plural, =0 {No partners} one {# partner} other {# partners}}
```

---

## 6. Regional formats (CLDR-class)

Per locale (and tenant override):

| Area | Examples |
|---|---|
| Number | decimal/group separators, min/max fraction |
| Currency | symbol/code placement, accounting negative |
| Percent | pattern |
| Date/time | short/medium/long/full, 12h/24h |
| Calendar | gregory, indian (where supported) |
| Week | first day, weekend |
| Collation | sort locale |
| Address | line order patterns |
| Person name | given/family order |
| Phone | display pattern hints |
| Measurement | metric/imperial preference labels |

Formats are resolved independently from message text but share locale + layers.

---

## 7. Translation management (TMS)

### Lifecycle

```text
KEY_CREATED → QUEUED → IN_TRANSLATION → IN_REVIEW → APPROVED → PUBLISHED
                              ↘ REJECTED ↗
```

### Artifacts

- Jobs by locale/namespace  
- Assignments to translators/reviewers  
- Comments / issues  
- Screenshots / UI context URLs  
- TM matches (exact/fuzzy)  
- Glossary hits / violations  

### Publish

Approved set → immutable `i18n_publish_version` + compressed bundle artifact → outbox → cache invalidate.

---

## 8. Packs

Language/country packs (`i18n_package`):

- `hi-IN` UI pack  
- `ar-AE` RTL pack  
- India GST terminology pack  
- Legal disclaimer pack  

Install verifies checksum/signature hook; applies as LANGUAGE_PACK layer; can require approval.

---

## 9. Machine translation

- Providers configured via configuration secrets (`configuration.secret.*`)  
- `i18n_mt_run` stores proposals only  
- Policy: MT never auto-publishes to production without `i18n.mt.auto_publish` feature + review bypass permission  

---

## 10. Security

### Permissions

| Code | Use |
|---|---|
| `i18n.catalog.read` | Read published bundles / resolve |
| `i18n.catalog.manage` | Manage keys/namespaces (platform) |
| `i18n.translate` | Submit translations |
| `i18n.review` | Approve/reject translations |
| `i18n.publish` | Publish / rollback bundles |
| `i18n.pack.install` | Install language packs |
| `i18n.override.manage` | Tenant/company overrides |
| `i18n.glossary.manage` | Glossary |
| `i18n.tm.manage` | TM admin |
| `i18n.mt.run` | Run MT proposals |
| `i18n.coverage.read` | Coverage reports |
| `i18n.pseudo.run` | Pseudo-loc generation |
| `i18n.audit.read` | Audit |
| `i18n.*` | Wildcard |

### RLS

Tenant overrides / jobs / comments: FORCE RLS by `tenant_id`.  
System catalog readable with auth context.

---

## 11. Module layout

```text
platforms/p06_localization/
  application/
    services/
      effective_resolver.py
      icu_engine.py
      fallback_chain.py
      publisher.py
      coverage_scanner.py
      tm_matcher.py
      glossary_linter.py
      package_installer.py
      mt_orchestrator.py
      pseudo.py
    commands/… queries/…
    permissions/catalog.py
    errors.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/outbox/
    adapters/mt_*.py  xliff.py
  tests/unit/icu/ resolver/ tms/
```

Load after configuration (registry deps 01, 03); typically after metadata in app composition when both Live.

---

## 12. Integration rules

1. **Never hardcode UI English** in frontend for cataloged keys — resolve via bundle or SSR API.  
2. Metadata `label_key` → i18n resolve.  
3. Notification templates store keys + ICU, not baked Hindi.  
4. User locale preference from IAM profile; company default from ORG/config.  
5. PDF generators pass locale into resolve.  
6. Gateways only — no ORM imports from other platforms.  
7. Cache bundles by `(tenant, company, locale, channel, version)`.

---

## 13. Domain events

| Event | When |
|---|---|
| `i18n.bundle.published` / `rolled_back` | Publish |
| `i18n.pack.installed` / `uninstalled` | Packs |
| `i18n.key.created` / `deprecated` | Catalog |
| `i18n.translation.approved` | TMS |
| `i18n.override.changed` | Tenant override |
| `i18n.coverage.failed` | Gate failed |
| `i18n.mt.completed` | MT batch done |

Stream: `jesloterp:localization:outbox`.

---

## 14. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, locale/language seeds, permissions |
| P2 | Namespaces/keys/messages + ICU engine |
| P3 | Regional formats + fallback resolve + ETag bundles |
| P4 | TMS workflow + glossary + TM |
| P5 | Packs + publish/rollback |
| P6 | Overrides + coverage + pseudo-loc |
| P7 | XLIFF import-export + MT adapters |
| P8 | Registry → **Live** | ✅ |

---

## 15. Definition of Done (enterprise)

- [x] ICU plural/select tests for `en`, `hi`, `ar`  
- [x] Fallback chain unit tests  
- [x] Layer provenance on resolve debug  
- [x] Published bundle immutability + checksum  
- [x] Glossary lint blocks forbidden terms  
- [x] Coverage gate for critical namespaces  
- [x] RTL `direction` returned for `ar-*`  
- [x] Tenant override RLS (ORM + policy; migration via parent)  
- [x] MT proposals cannot publish without policy  
- [x] XLIFF round-trip for one namespace (HTTP export/import + adapter stub)  
- [x] Tenant RLS GUCs on HTTP (`require_i18n_access`)  
- [x] Messages + ICU overlays persist on `AsyncSession` (empty catalog is `[]`)  
- [x] No cross-schema FKs  

---

## 16. Anti-patterns

| Don’t | Do |
|---|---|
| Concatenate sentences in code | ICU messages |
| Ship only `en` JSON in frontend | Resolve bundles |
| Store Hindi inside Metadata fields | `label_key` + i18n |
| Auto-publish raw MT | Review/approve policy |
| One flat string table without namespaces | Namespaced catalogs |
| Ignore plurals for Hindi/Arabic | ICU plural rules |
| Trust `Accept-Language` alone for authz | Still use JWT tenant |

---

## 17. Related documents

- Schema: [`LOCALIZATION_SCHEMA.md`](LOCALIZATION_SCHEMA.md)  
- API: [`LOCALIZATION_API.md`](LOCALIZATION_API.md)  
- Metadata keys: [`../05_metadata/METADATA_GUIDE.md`](../05_metadata/METADATA_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
