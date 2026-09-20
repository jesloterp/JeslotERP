# Multi-tenancy

## Actual model

JeslotERP is multi-tenant.

- `p02_organization` is the tenant / company / branch system of record.
- `p01_identity` binds users to tenant context and supports context switch.
- Tenant-owned tables use PostgreSQL row-level security and fail closed.
- Sharing (`p33_sharing`) is record ACL **inside** a tenant, not a way to
  escape tenancy.

## Scopes

Typical scope stack:

```text
Tenant
  Company
    Branch / location
      Record ACL (teams, rules, grants)
```

Configuration and metadata resolve across tenant and company layers.

## What tenancy is not

- It is not a separate database per tenant in the public architecture (that
  deployment choice is private and unpublished).
- It is not IAM roles alone.
- It is not licensing SKUs (those gate commercial capability).

## Future

Business modules must set RLS context the same way platform packages do.
No cross-tenant UUID fetch without an explicit, audited break-glass path.
