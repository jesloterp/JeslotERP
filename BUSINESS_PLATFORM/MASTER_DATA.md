# Master Data

## Implemented on the platform (not business packages)

| Master | Package | Status |
|---|---|---|
| User, role, permission | `p01_identity` | IMPLEMENTED |
| Tenant, company, branch, fiscal, UoM, FX, warehouse location | `p02_organization` | IMPLEMENTED |
| Settings definitions | `p03_configuration` | IMPLEMENTED |
| Business partner and roles | `p04_business_partner` | IMPLEMENTED |

## Planned business masters

Items, BOMs, tax codes, house banks, work centers, equipment, WBS, cost
objects, service catalogs. See the TODO file.

## Rules

- Masters get numbers from `p07_number_series`.
- Masters get layouts from `p05_metadata`.
- Masters emit events; they do not update ledgers directly except through
  documented posting services.
