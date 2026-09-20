# Audit Architecture

**Package:** `p19_audit`  
**Status:** PARTIALLY_IMPLEMENTED

## Current capability

- Immutable audit event ingest and query.
- Field-change concept, streams, seals, retention, legal hold, export.
- Break-glass read paths.
- Internal ingest including an event-shaped entry point.
- SIEM HTTP port; fail-closed without an endpoint.

## Distinct from

| Concern | Owner |
|---|---|
| Technical logs | `p20_logging` |
| Metrics / SLO | `p21_monitoring` |
| Privacy DSR | `p31_privacy` (uses audit / hold primitives) |
| Identity security events | `p01_identity` emits; audit retains |

## Planned

- Live SIEM.
- Every business module emitting before-after field diffs on financial documents.
