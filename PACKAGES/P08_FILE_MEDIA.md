# File / Media (`p08_file_media`)

**Package:** `p08_file_media`  
**Schema:** `media`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Binary media control plane: upload, storage backends, scan / quarantine, attachments, and object lifecycle.

## 2. Responsibilities

- Own the `media` persistence schema and the `p08_file_media` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Direct, multipart, and proxy upload sessions
- Object CRUD and download
- Variant processing
- Scan orchestration
- Attachments and collections
- Admin backends, policies, and quotas
- Local storage path is implemented
- Cloud storage and antivirus adapters exist as ports and fail-closed without credentials

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Media object**
- **Blob**
- **Upload session**
- **Quarantine**
- **Variant / rendition**
- **Attachment link**
- **Storage backend**
- **Content policy**
- **Quota**

## 5. Major Capabilities

- Direct, multipart, and proxy upload sessions
- Object CRUD and download
- Variant processing
- Scan orchestration
- Attachments and collections
- Admin backends, policies, and quotas
- Local storage path is implemented
- Cloud storage and antivirus adapters exist as ports and fail-closed without credentials

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Storage config
- Objects / blobs
- Uploads
- Scan / quarantine
- Variants
- Links
- Governance
- Outbox

Public API resource groups:

- Uploads
- Objects
- Attachments
- Admin
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`

### Additional verified runtime coupling

- Record sharing via p33_sharing
- Auth contract shared with the metadata plane

### Recommended future dependency

- Consumed by p09_document, p27_ai corpora, p32_output render artifacts

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Document
- AI
- Output
- Reporting exports (recommended)
- Privacy erase adapters

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `media.upload.initialized`
- `media.upload.completed`
- `media.scan.clean`
- `media.scan.infected`
- `media.available`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Storage and scan providers attach at startup when configured; otherwise they remain pending.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- MIME and size policy
- Quarantine
- Legal hold
- Sharing / ACL
- media permissions
- RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Storage adapter port
- Scanner port
- Package seeds for backends

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `media` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Object-storage and antivirus live credentials are environment-blocked
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Contracts, identity documents, drawings, and generated PDFs all attach through this plane.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
