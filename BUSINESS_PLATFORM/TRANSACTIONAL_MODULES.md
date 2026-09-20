# Transactional Modules

Transactional modules create numbered documents that change economic state.

| Document (planned) | Module | Affects |
|---|---|---|
| Journal | `b01_finance` | GL |
| Sales order / delivery / AR invoice | `b05_sales` | Stock, AR, revenue |
| PR / PO / GR / AP invoice | `b06_purchasing` | Stock, AP, GR/IR |
| Stock movement | `b07_inventory` | On-hand, value |
| Production order | `b10_manufacturing` | Components, FG, WIP |
| Depreciation run | `b13_fixed_assets` | Expense, accum. dep. |
| Shipment | `b09_logistics` | Freight, delivery refs |

**None of these documents are implemented as business packages.**

Shared kernel services they must use: numbers, metadata, process, rules,
audit, sharing, output, events.
