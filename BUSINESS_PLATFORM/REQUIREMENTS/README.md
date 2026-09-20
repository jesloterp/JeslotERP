# Business Module Requirements

**Documentation status:** this is the first finished **requirements** set for
all seventeen registered modules. **Code status:** all `PLANNED` / `NOT_FOUND`.

**Speed:** implement in record-winning time — [../../DELIVERY_SPEED.md](../../DELIVERY_SPEED.md).
Do not re-open these specs unless a posting rule is wrong.

Planning method: [../HOW_TO_PLAN.md](../HOW_TO_PLAN.md).

## Recommended implementation order

| Wave | Module | Why this wave |
|---|---|---|
| 0 | Decision | Record first `bNN` |
| 1 | `b01_finance` | Books are economic law |
| 1b | `b07_inventory` | Stock truth; items |
| 2 | `b03_tax` | Calculate before invoices |
| 2 | `b05_sales` + `b06_purchasing` | Documents that post |
| 3 | `b04_treasury` + `b02_controlling` | Cash and cost |
| 4 | `b08_warehouse` + `b09_logistics` | After inventory |
| 5 | `b13_fixed_assets` + `b14_projects` | After finance |
| 6 | `b10_manufacturing` + `b11_quality` | After inventory + CO |
| 7 | `b16_crm` + `b12_maintenance` + `b17_service` + `b15_hcm` | Downstream |

## Requirement files

| ID | File | Package |
|---|---|---|
| B01 | [B01_FINANCE.md](B01_FINANCE.md) | `b01_finance` |
| B02 | [B02_CONTROLLING.md](B02_CONTROLLING.md) | `b02_controlling` |
| B03 | [B03_TAX.md](B03_TAX.md) | `b03_tax` |
| B04 | [B04_TREASURY.md](B04_TREASURY.md) | `b04_treasury` |
| B05 | [B05_SALES.md](B05_SALES.md) | `b05_sales` |
| B06 | [B06_PURCHASING.md](B06_PURCHASING.md) | `b06_purchasing` |
| B07 | [B07_INVENTORY.md](B07_INVENTORY.md) | `b07_inventory` |
| B08 | [B08_WAREHOUSE.md](B08_WAREHOUSE.md) | `b08_warehouse` |
| B09 | [B09_LOGISTICS.md](B09_LOGISTICS.md) | `b09_logistics` |
| B10 | [B10_MANUFACTURING.md](B10_MANUFACTURING.md) | `b10_manufacturing` |
| B11 | [B11_QUALITY.md](B11_QUALITY.md) | `b11_quality` |
| B12 | [B12_MAINTENANCE.md](B12_MAINTENANCE.md) | `b12_maintenance` |
| B13 | [B13_FIXED_ASSETS.md](B13_FIXED_ASSETS.md) | `b13_fixed_assets` |
| B14 | [B14_PROJECTS.md](B14_PROJECTS.md) | `b14_projects` |
| B15 | [B15_HCM.md](B15_HCM.md) | `b15_hcm` |
| B16 | [B16_CRM.md](B16_CRM.md) | `b16_crm` |
| B17 | [B17_SERVICE.md](B17_SERVICE.md) | `b17_service` |

Each file follows the same section contract so a serious developer can pick
up any module and know what “done” means.
