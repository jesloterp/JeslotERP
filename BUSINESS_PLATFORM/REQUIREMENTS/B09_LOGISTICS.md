# Logistics / Transportation — Requirement Specification

| Field | Value |
|---|---|
| ID | `B09` |
| Package | `b09_logistics` |
| Schema | `logistics` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can deliveries or independent consignments be planned onto a trip with a carrier partner and optional freight accrual?

After sales/purchase/inventory. Not a vertical transport product in b01–b17.

## 2. In scope

- Consignment / shipment document
- Trip with resource or carrier assignment
- Status events (departed / arrived)
- Freight estimate/actual interface to finance and optional customer invoice

## 3. Out of scope

- Industry-specific fields (overlays later, not kernel)
- Public map products
- Fleet IoT

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Planner | `logistics.shipment.write` | Create shipment |
| Dispatcher | `logistics.trip.dispatch` | Assign resource |

Permission codes are namespaced (`logistics.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Vehicle / resource (thin)** — Optional master
- **Service level** — Transit terms

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `logistics.shipment` | Shipment | `planned → in_transit → delivered/cancelled` | `logistics.ship` | Optional freight accrual |
| `logistics.trip` | Trip | `planned → completed` | `logistics.trip` | None |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Shipment links to source deliveries by UUID; no cross-schema FK.
- Complete trip may require a resource when policy says so.
- Freight accrual journal is idempotent per shipment.

## 8. Processes and rules

- **logistics.exception** — Failed delivery / detention

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `logistics.shipment.departed`
- `logistics.shipment.delivered`
- `logistics.freight.accrued`

### Reports (`p24_reporting` datasets — planned content)

- In-transit shipments
- Freight accrual register

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Shipment linked to one delivery + statuses |
| Advanced | Multi-stop trip + freight invoice |

## 14. Acceptance criteria (MVP)

- Shipment cannot complete without required status events.
- Horizontal fields only; vertical fields are overlays.


## 10. Platform dependency map

Must consume, not reimplement: `p01_identity`, `p02_organization`,
`p03_configuration`, `p05_metadata`, `p07_number_series`, `p10_process`,
`p11_rules`, `p13_event_bus`, `p19_audit`, `p32_output`, `p33_sharing`.

Optional when the document type needs them: `p04_business_partner`,
`p08_file_media`, `p09_document`, `p14_messaging`, `p15_notification`,
`p17_scheduler`, `p24_reporting`, `p26_licensing`.

## 11. Non-claims (independent product)

- Do not copy another product's table names or document tree.
- Do not hide posting in a website controller.
- Do not ship a document that only prints.
- Do not start this module before its predecessor wave is specified.

## 12. Documentation finish checklist

- [x] Purpose and economic question written
- [x] Documents and integrity rules named
- [x] MVP acceptance criteria testable
- [ ] First module chosen by product owner (Chetan Patel)
- [ ] Implementation in private code (not this repository)
