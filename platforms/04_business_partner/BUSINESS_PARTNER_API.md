# JeslotERP Business Partner Platform — Complete API Specification (Advanced)

**Version:** 2.0  
**Last reviewed:** 2026-09-09  
**Status:** **Live** — complete public `/api/v1/bp` + internal `/internal/v1/bp` surface  
**Package:** `platforms.p04_business_partner`  
**Public base:** `/api/v1/bp`  
**Internal base:** `/internal/v1/bp`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`BUSINESS_PARTNER_GUIDE.md`](BUSINESS_PARTNER_GUIDE.md) · [`BUSINESS_PARTNER_SCHEMA.md`](BUSINESS_PARTNER_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-09 | CRUD partners + satellites. |
| **2.0** | **2026-09-09** | Validate-for-use, match/merge, KYC, change requests, sites, aliases, external IDs, consent, erasure, risk, graph, export. |
| **2.1** | **2026-09-12** | P33-LIVE-002: GET `/partners` share-filters when grants exist. |
| **2.2** | **2026-09-12** | P33-LIVE-003: partner-scoped public GETs + validate-for-use evaluate p33. |

---

## 1. Design principles (advanced)

1. JWT/context-switch is sole tenant authority.  
2. Default reads mask PII (bank/PAN/Aadhaar).  
3. `validate-for-use` is the gate for bilty/invoice/PO create.  
4. Sensitive field changes may require change-request approval.  
5. Merge/erasure are governed, audited, outboxed.  
6. Idempotent partner create; OCC on partner root.  
7. Soft-delete / MERGED — no silent hard delete.  
8. Internal mesh never uses user JWT.  
9. List/search support role + match-key filters.  
10. `custom_fields` validated via p05 when available.

---

## 2. Headers

```http
Authorization: Bearer <token>
Idempotency-Key: <key>          # create, merge, import
If-Match: <version>             # patch partner
X-Request-ID: <uuid>
```

Reveal: `?reveal=true` requires manage permission for that PII class + is audited.

---

## 3. Errors (extended)

```text
BP_PARTNER_NOT_FOUND / CODE_EXISTS / INACTIVE / BLACKLISTED / LEGAL_HOLD
BP_ROLE_MISSING / ROLE_NOT_EFFECTIVE
BP_COMPANY_SCOPE_DENIED
BP_VALIDATION_FAILED
BP_MATCH_NOT_FOUND / MERGE_CONFLICT / MERGE_BLOCKED
BP_KYC_STATE_INVALID
BP_CHANGE_REQUEST_REQUIRED / APPROVAL_REJECTED
BP_ERASURE_BLOCKED
BP_EXTERNAL_ID_CONFLICT
BP_VERSION_CONFLICT / IDEMPOTENCY_CONFLICT
BP_PERMISSION_DENIED / FLS_DENIED
```

---

## 4. Permissions

See Guide §10. Route matrix uses the same codes (`bp.partner.*`, `bp.match.manage`, `bp.kyc.manage`, `bp.partner.erase`, …).

---

## 5. Partners (core)

Base: `/api/v1/bp/partners`

| Method | Path | Permission |
|---|---|---|
| GET | `/partners` | read |
| POST | `/partners` | create (+ Idempotency-Key) |
| GET | `/partners/{id}` | read |
| PATCH | `/partners/{id}` | update (may open change-request) |
| DELETE | `/partners/{id}` | delete |
| POST | `/partners/{id}/activate` \| `/deactivate` | update |
| POST | `/partners/{id}/blacklist` \| `/unblacklist` | blacklist |
| GET | `/partners/{id}/timeline` | read — audit+events projection |
| GET | `/partners/{id}/effective` | read — roles/scopes/blocks summary |

### List filters (advanced)

`search`, `role`, `partner_type`, `category_code`, `kyc_status`, `is_blacklisted`, `company_id`, `territory_id`, `segment_id`, `gstin`, `pan`, `external_system`, `external_key`, `match_status`, `quality_min`.

List also share-filters: a row with no p33 grants stays visible (same skip as GET-by-id). A row with grants is dropped when evaluate denies. Filter is page-local (`total` reduced by dropped rows on this page). Not a SQL share predicate.

Public partner-scoped GETs (satellites, roles, credit, sites, notes, onboarding, …) and `POST /partners/{id}/validate-for-use` call `require_partner_share`. No grants → skip. Grants + deny → 403 `SHR_FORBIDDEN`. Internal `/internal/v1/bp` is unchanged.

### Create body (advanced)

Includes v1 fields plus: `aliases[]`, `roles[]`, `sites[]`, `external_ids[]`, `identifiers` auto-derived, `segment_id`, `territory_id`, `custom_fields`, `start_onboarding: true`.

---

## 6. Validate-for-use (critical)

### Public (user)

```http
POST /api/v1/bp/partners/{id}/validate-for-use
{ "required_role": "CONSIGNOR", "company_id": null, "as_of": "2026-09-09" }
```

### Internal (mesh)

```http
POST /internal/v1/bp/partners/validate-for-use
X-Internal-Token: …
{
  "tenant_id": "…",
  "partner_id": "…",
  "company_id": "…",
  "required_role": "CONSIGNOR",
  "as_of": "2026-09-09"
}
```

### Response

```json
{
  "valid": false,
  "partner_id": "…",
  "display_name": "Acme",
  "checks": [
    { "code": "EXISTS", "ok": true },
    { "code": "ACTIVE", "ok": true },
    { "code": "ROLE", "ok": true },
    { "code": "COMPANY_SCOPE", "ok": true },
    { "code": "NOT_BLOCKED", "ok": false, "detail": "BLACKLIST" }
  ]
}
```

Transport/Finance **must** call this before creating documents.

---

## 7. Roles & scope

Same as v1 `/roles`, `/companies`, `/branches` plus:

- dated effectiveness on assign  
- `POST /roles/bulk`  
- `GET /partners/{id}/roles/effective?as_of=`  

---

## 8. Aliases, names, identifiers, external IDs

| Method | Path | Permission |
|---|---|---|
| GET/POST/DELETE | `/partners/{id}/aliases` | update |
| GET/PUT | `/partners/{id}/person-name` | update |
| GET | `/partners/{id}/identifiers` | read |
| POST | `/partners/{id}/identifiers/rebuild` | update — recompute norms |
| GET/POST/DELETE | `/partners/{id}/external-ids` | update |

```json
{ "system_code": "GSTN", "external_key": "27ABCDE1234F1Z5" }
```

---

## 9. Sites & addresses & contacts

### Sites — `/partners/{id}/sites`

Full CRUD + `POST /sites/{site_id}/default-shipping`.

### Addresses / contacts

v1 item CRUD is shipped:

| Method | Path |
|---|---|
| GET/POST | `/partners/{id}/addresses` |
| GET/PUT/DELETE | `/partners/{id}/addresses/{address_id}` |
| GET/POST | `/partners/{id}/contacts` |
| GET/PUT/DELETE | `/partners/{id}/contacts/{contact_id}` |

Contacts also have `/contacts/{contact_id}/roles` and `/preferences`. Match/merge scan + journal exist; **no MDM steward / golden-record workflow**.

### Consent

```http
GET/POST /partners/{id}/consents
POST /partners/{id}/consents/{consent_id}/revoke
```

Permission: `bp.contact.manage` (or dedicated later).

### Account team

```http
GET/PUT /partners/{id}/account-team
```

---

## 10. Tax, identity, bank, verification

v1 tax/identity/bank CRUD **plus**:

| Method | Path | Permission |
|---|---|---|
| POST | `/tax-registrations/{id}/verify` | tax.manage |
| POST | `/bank-accounts/{id}/verify` | bank.manage |
| GET | `/partners/{id}/verifications` | read |
| GET | `/bank-accounts/{id}?reveal=true` | bank.manage |

---

## 11. Relationships & hierarchy

| Method | Path |
|---|---|
| GET/POST | `/partners/{id}/relationships` |
| GET | `/partners/{id}/hierarchy` | ancestors/descendants via closure |
| GET | `/partners/{id}/network?depth=2` | bounded graph |

---

## 12. Credit, payment, risk

| Method | Path | Permission |
|---|---|---|
| GET/PUT | `/partners/{id}/credit-profile` | credit.manage |
| GET/PUT | `/partners/{id}/payment-profile` | credit.manage |
| GET | `/partners/{id}/risk-scores` | read |
| POST | `/partners/{id}/risk-scores` | credit.manage |
| GET | `/partners/{id}/exposure` | read — snapshot only |

---

## 13. Compliance

| Method | Path | Permission |
|---|---|---|
| GET/POST | `/partners/{id}/compliance-flags` | blacklist/update |
| POST | `/compliance-flags/{id}/clear` | blacklist |
| POST | `/partners/{id}/legal-hold` | blacklist/approve |
| POST | `/partners/{id}/legal-hold/release` | approve |

---

## 14. KYC / onboarding

| Method | Path | Permission |
|---|---|---|
| POST | `/partners/{id}/onboarding/start` | kyc.manage |
| GET | `/partners/{id}/onboarding` | read |
| POST | `/onboarding/{case_id}/submit` | kyc.manage |
| POST | `/onboarding/{case_id}/checklist/{item_id}/complete` | kyc.manage |
| POST | `/onboarding/{case_id}/approve` | partner.approve |
| POST | `/onboarding/{case_id}/reject` | partner.approve |

---

## 15. Change requests & approvals

| Method | Path | Permission |
|---|---|---|
| POST | `/partners/{id}/change-requests` | update |
| GET | `/change-requests` | approve/read |
| POST | `/change-requests/{id}/submit` | update |
| POST | `/change-requests/{id}/approve` | partner.approve |
| POST | `/change-requests/{id}/reject` | partner.approve |
| POST | `/change-requests/{id}/apply` | approve/update |

Policy (config): which fields require CR (legal_name, bank, primary GSTIN, …).

---

## 16. Match & merge

| Method | Path | Permission |
|---|---|---|
| POST | `/match/scan` | match.manage — scan tenant/partners |
| GET | `/match/candidates` | match.manage |
| POST | `/match/candidates/{id}/dismiss` | match.manage |
| POST | `/match/candidates/{id}/confirm` | match.manage |
| POST | `/partners/merge` | partner.merge |
| GET | `/merge-journals/{id}` | match/merge read |
| POST | `/merge-journals/{id}/reverse` | partner.merge (rare; guarded) |

### Merge body

```json
{
  "survivor_partner_id": "UUID",
  "victim_partner_id": "UUID",
  "survivorship_policy_key": "DEFAULT",
  "dry_run": false
}
```

Dry-run returns planned satellite moves without committing.

---

## 17. Erasure (privacy)

| Method | Path | Permission |
|---|---|---|
| POST | `/partners/{id}/erasure-requests` | partner.erase |
| GET | `/erasure-requests` | erase/approve |
| POST | `/erasure-requests/{id}/approve` | partner.approve |
| POST | `/erasure-requests/{id}/execute` | partner.erase |

If `legal_hold` → `409 BP_ERASURE_BLOCKED`.  
Execute anonymizes PII; keeps `id` for FK integrity; emits `bp.partner.erasure_completed`.

---

## 18. Notes, attachments, audit, export

| Method | Path | Permission |
|---|---|---|
| CRUD | `/partners/{id}/notes` | update/read |
| CRUD | `/partners/{id}/attachments` | update/read |
| GET | `/partners/{id}/audit` | read |
| POST | `/partners/export` | bp.export |
| POST | `/partners/import` | create (+ match optional) |

Import supports dry-run + dedupe report.

---

## 19. Lookups / settings

Tenant: `/categories`, `/classifications`, `/groups`, `/segments`, `/territories`, `/payment-terms`, `/industries`.  
System: `/lookups/*`, `/settings/*` with `bp.settings.manage`.

---

## 20. Internal mesh API

| Method | Path | Purpose |
|---|---|---|
| POST | `/partners/validate-for-use` | Gate documents |
| GET | `/partners/{id}` | Resolve |
| GET | `/partners/by-code` | `tenant_id+code` |
| GET | `/partners/by-external-id` | system+key |
| POST | `/partners/bulk-resolve` | ids → summaries |
| GET | `/partners/{id}/primary-gstin` | Quick tax |
| POST | `/partners/match` | Online match suggest |
| GET | `/health` | Liveness |

---

## 21. Example flows

### 21.1 Onboard transporter

1. `POST /partners` roles `[VENDOR,TRANSPORTER]`, `start_onboarding: true`  
2. Tax + bank + documents  
3. Submit KYC → approve  
4. Activate  

### 21.2 Create bilty safely

1. Internal `validate-for-use` role `CONSIGNOR`  
2. If valid → create LR with `partner_id` + snapshot  

### 21.3 Duplicate GSTIN

1. Create blocked by identifier unique **or** match candidate opened  
2. Reviewer confirms → `POST /partners/merge`  
3. Consumers handle `bp.partner.merged`  

---

## 22. Implementation checklist (v2)

- [x] validate-for-use (public + internal) with block matrix  
- [x] Identifier maintenance + unique norms  
- [x] Mask/reveal + audit  
- [x] KYC case + approvals  
- [x] Change-request policy  
- [x] Match scan + merge journal + outbox  
- [x] Erasure + legal hold  
- [x] Sites/aliases/external IDs  
- [x] FORCE RLS + contract tests  
- [x] Retarget legacy `p03_business_partner` imports  

---

## 23. Boundary reminders

| Need | Where |
|---|---|
| Render partner form | p05 `/metadata/ui-pack?entity_key=bp.partner` |
| Partner row / roles | p04 `/bp` |
| Setting values | p03 |
| File bytes | p08 via `media_id` |
| Translations | p06 |
