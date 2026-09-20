# Metadata Platform

**Package:** `p05_metadata`  
**Status:** IMPLEMENTED (kernel) — Production label withheld

This is the public specification of JeslotERP's metadata control plane.

## Current capability versus target capability

| Area | Current capability | Target capability |
|---|---|---|
| Entities and fields | Dictionary of modules, entities, fields, options, relations | Same, plus every business document seeded |
| Layered resolution | SYSTEM → PACK → TENANT → COMPANY → ROLE → USER → CHANNEL → FEATURE_FLAG | Unchanged model, more overlays in the wild |
| Validation | Server validate API; conditional required / visible / readonly | Shared expression conformance with every client |
| UI metadata | Forms, lists, filters, actions, inspectors, variants, user prefs persist | Generic `/desk/.../e/:entityKey` for any seeded entity |
| UI pack | Effective pack with ETag / freshness comparison | Studio-first authoring as the default operator path |
| Field security | Classification, mask, required permission descriptors | Enforcement on every bound API |
| Publish | Changesets, approve, publish, rollback, impact, drift | ALM-transported packs as the normal promote path |
| API metadata | Binding / interop hints per entity | Generated public clients |
| Workflow metadata | Form keys referenced by process; not a BPMN store | Process forms fully described by metadata |
| Reporting metadata | Recommended consumer of the dictionary | Datasets auto-projected from entities |
| Dynamic app creation | **Not claimed.** Pilots: `bp.partner`, `org.company` | New masters/documents are seed + permission |

Metadata detects drift against physical schema. It must **not** silently rewrite production DDL.

## Core ideas

1. **Contract catalog.** Typed domain models remain the physical truth. Metadata describes them.
2. **Configuration ≠ metadata.** Configuration stores values. Metadata stores shape, UI, and rules.
3. **Safe AST.** Visibility, defaults, and computed hints use allow-listed operators. No `eval`, no SQL.
4. **Key stability.** Entity and field keys are stable after first publish; deprecate + replace.
5. **Extensibility without fork.** Tenants add overlays; they cannot delete system fields.
6. **Publish immutability.** Runtime pins a published artifact; rollback republishes.

## How a business application can be created

Intended path (partly implemented):

1. Declare the entity and fields in the dictionary.
2. Author list / form / filter / action / inspector layouts.
3. Bind list sort / filter contracts.
4. Publish a version.
5. Allocate numbers via `p07_number_series`.
6. Persist rows in the owning package (today: `p04_business_partner`, `p02_organization`; tomorrow: `business/bNN_*`).
7. Attach process, rules, sharing, print, and audit.

**Do not claim** that an arbitrary ERP module can be created from metadata alone today. The generic entity data façade is still `PENDING`.

## Permissions and lifecycle

- `metadata.*` permission catalog.
- RLS on catalog access.
- Changesets submit → approve / reject → publish.
- Pack ETag drift alerts when a client is stale.

## Related packages

`p01_identity` (enforcement), `p06_localization` (label keys), `p12_feature` (layer),
`p29_extensibility` (publish hooks), `p11_rules` (related AST safety bar).
