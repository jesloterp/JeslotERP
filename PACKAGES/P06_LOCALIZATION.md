# Localization (`p06_localization`)

**Package:** `p06_localization`  
**Schema:** `i18n`  
**Layer:** Platform Services  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Internationalization and localization control plane: locales, ICU messages, formats, translation workflow, and language packs.

## 2. Responsibilities

- Own the `i18n` persistence schema and the `p06_localization` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Locale catalog
- Effective ICU message resolve
- Overrides
- Format profiles
- Glossary and translation memory
- Translation management tasks
- Quality / coverage
- Import / export
- Packages and publish
- Seeded default locales including an English plus additional Indic locales used by the partner pilot

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Locale**
- **Message key / ICU message**
- **Format profile**
- **Glossary**
- **Translation memory**
- **Language pack**
- **Coverage**

## 5. Major Capabilities

- Locale catalog
- Effective ICU message resolve
- Overrides
- Format profiles
- Glossary and translation memory
- Translation management tasks
- Quality / coverage
- Import / export
- Packages and publish
- Seeded default locales including an English plus additional Indic locales used by the partner pilot

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Locale catalog
- Messages
- Formats
- Overrides
- Glossary / TM
- TMS jobs
- Packs
- Outbox

Public API resource groups:

- Locales
- Catalog
- Effective resolve
- Overrides
- TMS
- Glossary / TM
- Quality
- Publish / packages
- Import / export

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p03_configuration`

### Additional verified runtime coupling

- HTTP auth contract shared with the metadata plane

### Recommended future dependency

- p05_metadata label_key integration
- p02_organization locale defaults
- p15_notification template locales

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Notification templates
- Output determination
- Metadata labels
- future business UIs

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- Localization outbox events are present; specific public type catalog is NOT_VERIFIED beyond plumbing

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Fallback chains and optional MT secret references via configuration.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- i18n permissions
- RLS
- Publish gates based on coverage policy

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Language packs
- XLIFF / JSON import-export

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `i18n` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Machine-translation providers are policy-gated and not a silent publish path
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

ERP documents, tax forms, and operator UIs must resolve labels and formats per locale.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
