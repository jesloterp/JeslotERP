# Open-Source Roadmap

The implementation is **private**. This documentation tree is the first
public-facing artifact.

**Owner intent (Chetan Patel):** raise market funds, pay serious developers,
and open-source the product **only if it is very good**. A source release is
a quality gate, not a calendar promise and not a request for unpaid timepass.

## Phase 1 — Documentation

- Publish this architecture tree (current work).
- Keep status vocabulary honest.
- Add a public changelog of architecture decisions (no private tickets).

**Status:** DOCUMENTATION READY for this tree; source not included.

## Phase 2 — Architecture Review

- External review of package boundaries and the 33-number freeze.
- Confirm no circular plugin dependencies.
- Confirm business modules remain outside the kernel.

## Phase 3 — Security Review

- Signed threat model for identity and tenancy.
- Secret-scanning of any future public repo.
- Remove leftover private brand strings, hosts, and sample credentials
  from anything that might be published.
- Decide KMS / IdP baselines that can be demonstrated without customer keys.

## Phase 4 — Public API Definition

- Freeze resource groups and error-code families that outsiders may depend on.
- Publish OpenAPI as a generated artifact **without** internal-only routes
  if those leak workers or admin break-glass.
- Version policy: what is stable vs experimental.

## Phase 5 — Example Applications

- One public sample tenant story: company + partner + (when it exists) one
  business document — using only public APIs.
- No customer data, no production snapshots.

## Phase 6 — Developer Experience

- How to run a disposable kernel (to be written only when source is public).
- How to add a metadata entity.
- How to write an allow-listed hook.
- How to add a business module without touching pNN internals.

## Phase 7 — Community Infrastructure

- Code of conduct, security disclosure address, issue templates.
- DCO or CLA decision.
- Discussion forum vs issues policy.

## Phase 8 — Public Release

- Choose a source license independently from the documentation license.
- Publish a subset or the whole kernel **intentionally**, only after at least
  one business ledger posts and a threat review is signed.
- Tag a version. Do not drip private history that contains secrets.

## Phase 9 — Community Contributions

- Paid maintainers review kernel changes.
- Accept metadata packs and documentation first.
- Accept connectors and renderers second.
- Unpaid timepass and drive-by rewrites are rejected.

## Phase 10 — Ecosystem

- Certified connector list.
- Industry pack registry.
- Training curriculum based on these specifications.

## Readiness verdict

**DOCUMENTATION READY** for architecture publication.

**NOT READY** for source-code open source.
