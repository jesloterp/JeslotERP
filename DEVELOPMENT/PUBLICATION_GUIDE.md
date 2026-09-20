# Publication Guide

Use this checklist before anything leaves the private estate.

```text
PRIVATE
├── Source Code
├── Production Infrastructure
├── Credentials
├── Private Algorithms
├── Customer Data
├── Internal Configuration
└── Sensitive Deployment Details

PUBLIC
├── Architecture
├── Specifications
├── Concepts
├── Package Documentation
├── Roadmap
├── Business Module TODO
├── Extension Model
└── Developer Documentation
```

## Always exclude

- Source files, ORM models, raw SQL, migration revision ids used as
  forensic fingerprints if they leak unpublished schema.
- Passwords, tokens, API keys, JWKS, connection strings.
- Private URLs, IPs, soak hostnames, landscape deploy URLs.
- Customer names, tenant identifiers from production, personal data.
- Proprietary pricing or unpublished commercial terms.
- Internal comments that name customers or incidents.
- Brand strings of unpublished private products.
- Third-party logos, partner badges, or adjective branding (“SAP-class”,
  “ERPNext-style”). Named products are nominative only — see
  [../TRADEMARKS.md](../TRADEMARKS.md).
- Threat-model worksheets that list exploitable private paths.

## Usually safe

- Package ids `p01_identity` … `p33_sharing`.
- Business ids `b01_finance` … `b17_service`.
- Schema **names** (`identity`, `finance`).
- Public event name patterns.
- Resource group names.
- Status vocabulary.
- Mermaid diagrams of declared dependencies.

## Review questions

1. Could this help an attacker who does not have the source?
2. Could this identify a customer?
3. Does this claim a feature that tests do not support?
4. Does this rename a canonical package?
5. Does this include a private hostname?

If uncertain: **do not publish**. Describe the capability at one level higher.

## This tree

`public-platform-docs/` is intended to be extractable as its own Git
repository. Do not copy the private implementation into that repository.
