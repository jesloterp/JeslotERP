# Business Module Dependencies

## Actual (today)

```text
p01_identity
   ↓
p02_organization
   ↓
p03_configuration
   ↓
p04_business_partner     ← only business-adjacent master that exists
   ↓
(no business ledgers)
```

Platform services already available for the first module: `p05_metadata`,
`p07_number_series`, `p08_file_media`, `p09_document`, `p10_process`,
`p11_rules`, `p13_event_bus`, `p19_audit`, `p32_output`, `p33_sharing`.

## Future (registry build order)

```mermaid
graph TD
    P[Platform kernel p01-p33] --> B01[b01_finance]
    B01 --> B02[b02_controlling]
    B01 --> B03[b03_tax]
    B03 --> B04[b04_treasury]
    B01 --> B13[b13_fixed_assets]
    B01 --> B14[b14_projects]
    P --> B07[b07_inventory]
    B07 --> B08[b08_warehouse]
    B08 --> B09[b09_logistics]
    B07 --> B05[b05_sales]
    B07 --> B06[b06_purchasing]
    B01 --> B05
    B01 --> B06
    B07 --> B10[b10_manufacturing]
    B02 --> B10
    B10 --> B11[b11_quality]
    B07 --> B12[b12_maintenance]
    P --> B15[b15_hcm]
    P --> B16[b16_crm]
    B05 --> B16
    B16 --> B17[b17_service]
    B12 --> B17
```

All edges above except those into the kernel are **future dependencies**.

## Recommended first spine

```text
Identity → Organization → Business Partner → Items (b07)
    → Inventory ledger (b07) → Sales / Purchase (b05 / b06)
    → Accounting (b01) → Tax (b03) → Reporting (p24 content)
```

Alternative finance-first spine:

```text
Identity → Organization → COA / Period / Journal (b01)
    → Business Partner subledgers → Inventory / Sales later
```

## Rules

- Never reverse-depend: platforms must not import `business.bNN_*`.
- Shared documents (invoice that is both AR and tax) coordinate via events
  and posting APIs, not cross-schema foreign keys.
