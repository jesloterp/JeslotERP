# Feature (`p12_feature`)

**Package:** `p12_feature`  
**Schema:** `feature`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Feature-flag and staged-rollout control plane: targeting, kill switches, overrides, and experiments.

## 2. Responsibilities

- Own the `feature` persistence schema and the `p12_feature` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Flag and variation catalog
- Environments, rules, fallthrough, and prerequisites
- Segments
- Evaluate and bootstrap APIs
- Kill / override / break-glass
- Schedules, promote, experiments, packs, SDK keys
- Flag / kill / override HTTP is persistence-backed

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Flag**
- **Variation**
- **Environment**
- **Targeting rule**
- **Segment**
- **Kill switch**
- **Override**
- **Experiment**
- **SDK key**

## 5. Major Capabilities

- Flag and variation catalog
- Environments, rules, fallthrough, and prerequisites
- Segments
- Evaluate and bootstrap APIs
- Kill / override / break-glass
- Schedules, promote, experiments, packs, SDK keys
- Flag / kill / override HTTP is persistence-backed

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

- Catalog
- Variations / rules
- Segments / overrides / kills
- Experiments
- Schedules
- Governance
- Outbox

Public API resource groups:

- Flags
- Targeting
- Segments
- Evaluate / bootstrap
- Kills / overrides
- Governance
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`

### Additional verified runtime coupling

- Auth contract shared with the metadata plane

### Recommended future dependency

- p26_licensing entitlement compile
- p16_cache
- p17_scheduler for scheduled changes

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- API platform
- Dashboard
- frontend navigation gates
- metadata effective layer

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `feature.flag.created`
- `feature.kill.engaged`
- `feature.override.changed`
- `feature.experiment.started`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Compiled-store and in-process catalog; no secrets are documented here.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- feature permissions
- RLS
- Break-glass paths
- Eval audit

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Webhooks
- SDK keys
- Feature packs

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `feature` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Evaluate / bootstrap remain on an in-process catalog for part of the path
- License compile binding with p26_licensing is blocked pending product decision
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

New ERP modules and risky changes roll out by flag, not by emergency deploy.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
