# Configuration Architecture

**Package:** `p03_configuration`  
**Status:** IMPLEMENTED (kernel)

## Current capability

- Hierarchical definitions and scoped values (tenant / company / branch).
- Effective resolve.
- Secret **references** (ciphertext metadata; no plaintext public GET).
- Templates and dependency policies.
- Admin versus tenant APIs.
- Optional wrap via `p28_security` (local-dev verified; enterprise KMS pending).

## Distinct from

| Concern | Owner |
|---|---|
| Field shape / UI / validation | `p05_metadata` |
| Feature rollout | `p12_feature` |
| Commercial entitlement | `p26_licensing` |
| Org structure | `p02_organization` |

## Rule for business modules

Store policy values (tolerance, default tax code pointer, posting lock flags)
as configuration definitions. Do not hard-code them and do not hide them
inside metadata field defaults unless they are really UI defaults.
