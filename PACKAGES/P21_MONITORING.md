# Monitoring (`p21_monitoring`)

**Package:** `p21_monitoring`  
**Schema:** `monitoring`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Metrics, SLO / error budgets, probes, alerts, monitoring dashboards, and synthetics.

## 2. Responsibilities

- Own the `monitoring` persistence schema and the `p21_monitoring` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Status, probes, metrics, query / range
- SLI / SLO, alerts, routes, silences
- Monitoring dashboards, synthetics, incidents, trace policies
- Internal probe tick, alert / SLO eval, remote-write
- Metric catalog is persistence-backed
- Sample store uses an in-process engine unless an external TSDB attaches

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Metric definition**
- **Scrape target**
- **Probe**
- **SLI / SLO**
- **Alert rule**
- **Silence**
- **Synthetic check**
- **Incident**
- **Trace policy**

## 5. Major Capabilities

- Status, probes, metrics, query / range
- SLI / SLO, alerts, routes, silences
- Monitoring dashboards, synthetics, incidents, trace policies
- Internal probe tick, alert / SLO eval, remote-write
- Metric catalog is persistence-backed
- Sample store uses an in-process engine unless an external TSDB attaches

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

- Topology
- Metrics / scrape
- SLO / probes / traces
- Alerts / routes
- Dashboards / incidents
- Governance

Public API resource groups:

- Status / probes / metrics / SLO / alerts / dashboards / synthetics
- Internal eval

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p20_logging`

### Additional verified runtime coupling

- Identity permission seed; no direct logging-package import was verified

### Recommended future dependency

- All platforms as metric sources
- p15_notification for alert routing
- p13_event_bus

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Licensing metrics bridge
- operators

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `monitoring.probe.down`
- `monitoring.slo.burn_high`
- `monitoring.alert.firing`
- `monitoring.alert.resolved`
- `monitoring.synthetic.failed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

External TSDB attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- monitoring permissions
- Alert ack / resolve permissions
- RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- TSDB engine port
- Internal eval ticks

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `monitoring` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Prometheus / remote TSDB is provider-pending
- Production alert routing is not claimed

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

ERP availability and posting latency need SLOs, not only infrastructure ping.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
