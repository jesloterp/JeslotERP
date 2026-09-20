# Platform Principles

These principles are derived from the current kernel. They are design law for
future business modules and for any future public contributors.

## 0. Record-winning elapsed time

Chetan Patel will not wait out a long, drifting programme. Delivery must be
**short**. Speed comes from reusing p01–p33, following finished requirement
specs, and staffing one critical path — not from skipping posting integrity
and not from starting every module at once. See [DELIVERY_SPEED.md](DELIVERY_SPEED.md).

## 1. Document what exists

Never present a registry idea as a shipped ledger. Status words are defined in
the README. When evidence is thin, write `NOT_VERIFIED`.

## 2. One owner per concern

| If you need… | Ask… | Do not ask… |
|---|---|---|
| Who is the user? | `p01_identity` | Organization employee as login |
| Which company? | `p02_organization` | Identity tenant claims as the org master |
| What is the setting value? | `p03_configuration` | Metadata |
| What is the field shape / UI? | `p05_metadata` | Configuration |
| What number is next? | `p07_number_series` | Application `MAX(doc_no)+1` |
| Where are the bytes? | `p08_file_media` | Document or output |
| What is the document identity? | `p09_document` | Media |
| Who approves? | `p10_process` | A boolean column |
| What is the decision? | `p11_rules` | Process or domain `if` soup |
| Did it happen? | `p13_event_bus` + `p19_audit` | Shared mutable tables |
| May this SKU run? | `p26_licensing` | Feature flags as billing |
| May this row be seen? | `p33_sharing` + RLS | Role explosion |

## 3. Hexagonal packages

Application code depends on ports. Infrastructure adapters implement them.
HTTP routers stay thin. Other packages never import ORM models.

## 4. Fail closed

Missing providers, missing grants, missing entitlements, and missing RLS
context deny the action. Tests may use stubs; production must not invent
success identifiers.

## 5. Safe computation

Metadata visibility / defaults and rules expressions use allow-listed ASTs.
Extensibility handlers are allow-listed and isolated. Stored definitions must
not become arbitrary code execution.

## 6. Publish, do not mutate live contracts casually

Metadata, rules, localization, and feature configuration publish versions.
Rollback republishes a previous artifact.

## 7. Events over ORM coupling

Business facts travel as named events. Local outboxes exist today; the
intended distribution fabric is `p13_event_bus` plus `p14_messaging`.

## 8. Multi-tenant first

Tenant, company, and branch are first-class. PostgreSQL RLS is the isolation
primitive. Sharing is record ACL on top, not instead, of tenancy.

## 9. Business modules consume the kernel

`business/bNN_*` may depend on `pNN_*`. The reverse is forbidden.

## 10. Production is a gate, not a feeling

SoR-Live means the shipped HTTP slice persists. Production additionally
requires soak evidence, signed threat review, runbooks, and live-provider
honesty. Pytest alone never flips that label.
