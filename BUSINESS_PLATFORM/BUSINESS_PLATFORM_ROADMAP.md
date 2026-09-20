# Business Platform Roadmap

## Phase A — Decision and spine

1. Choose the first `bNN` (recommended: `b01_finance`).
2. Treat [REQUIREMENTS/B01_FINANCE.md](REQUIREMENTS/B01_FINANCE.md) as the build contract.
3. Ship plugin skeleton + metadata entity + number series + audit events.
4. Prove one document posts or one master+ledger works end to end.

## Phase B — Economic core

- If finance-first: COA → journal → period → trial balance dataset.
- If inventory-first: item → stock ledger → adjustment → valuation dataset.
- Then connect partner subledgers.

## Phase C — Order to cash / procure to pay

- Sales quote → order → delivery → AR invoice.
- PR → PO → GR → AP invoice + three-way match.
- Both post through the finance interface.

## Phase D — Tax, treasury, warehouse

- Tax calculate + register.
- Payments and house banks.
- Bins and picks if warehouse is in scope.

## Phase E — Downstream

- Manufacturing, quality, assets, projects, CRM, service, HCM.

## Phase F — Industry packs

- Overlays and ALM only after two horizontal documents are SoR-Live.

## Priority law

Architectural importance, not preference:

1. Posting integrity (numbers, periods, balanced journals, immutable stock lines).
2. Partner and item masters.
3. Metadata UI so the second document is not a new frontend.
4. Process + rules for exceptions.
5. Tax before multi-country claims.
6. Warehouse / manufacturing only after inventory truth.
