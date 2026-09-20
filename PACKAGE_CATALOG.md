# Package Catalog

Canonical names are frozen. Do not rename `p05_metadata` to “metadata-engine”
in public material.

| # | Package | Title | Schema | Layer | Status |
|---|---|---|---|---|---|
| 01 | `p01_identity` | Identity | `identity` | Foundation | IMPLEMENTED |
| 02 | `p02_organization` | Organization | `org` | Foundation | IMPLEMENTED |
| 03 | `p03_configuration` | Configuration | `configuration` | Foundation | IMPLEMENTED |
| 04 | `p04_business_partner` | Business Partner | `bp` | Foundation | IMPLEMENTED |
| 05 | `p05_metadata` | Metadata | `metadata` | Services | IMPLEMENTED |
| 06 | `p06_localization` | Localization | `i18n` | Services | IMPLEMENTED |
| 07 | `p07_number_series` | Number Series | `number_series` | Services | IMPLEMENTED |
| 08 | `p08_file_media` | File / Media | `media` | Services | PARTIALLY_IMPLEMENTED |
| 09 | `p09_document` | Document | `document` | Services | PARTIALLY_IMPLEMENTED |
| 10 | `p10_process` | Process | `process` | Services | PARTIALLY_IMPLEMENTED |
| 11 | `p11_rules` | Rules | `rules` | Services | PARTIALLY_IMPLEMENTED |
| 12 | `p12_feature` | Feature | `feature` | Services | PARTIALLY_IMPLEMENTED |
| 13 | `p13_event_bus` | Event Bus | `event_bus` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 14 | `p14_messaging` | Messaging | `messaging` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 15 | `p15_notification` | Notification | `notification` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 16 | `p16_cache` | Cache | `cache` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 17 | `p17_scheduler` | Scheduler | `scheduler` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 18 | `p18_search` | Search | `search` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 19 | `p19_audit` | Audit | `audit` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 20 | `p20_logging` | Logging | `logging` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 21 | `p21_monitoring` | Monitoring | `monitoring` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 22 | `p22_api` | API | `api` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 23 | `p23_integration` | Integration | `integration` | Infrastructure | PARTIALLY_IMPLEMENTED |
| 24 | `p24_reporting` | Reporting | `reporting` | Services | PARTIALLY_IMPLEMENTED |
| 25 | `p25_dashboard` | Dashboard | `dashboard` | Services | PARTIALLY_IMPLEMENTED |
| 26 | `p26_licensing` | Licensing | `licensing` | Services | PARTIALLY_IMPLEMENTED |
| 27 | `p27_ai` | AI | `ai` | Services | PARTIALLY_IMPLEMENTED |
| 28 | `p28_security` | Security | `security` | Foundation | PARTIALLY_IMPLEMENTED |
| 29 | `p29_extensibility` | Extensibility | `extensibility` | Services | IMPLEMENTED |
| 30 | `p30_alm` | ALM | `alm` | Services | PARTIALLY_IMPLEMENTED |
| 31 | `p31_privacy` | Privacy | `privacy` | Services | PARTIALLY_IMPLEMENTED |
| 32 | `p32_output` | Output | `output` | Services | PARTIALLY_IMPLEMENTED |
| 33 | `p33_sharing` | Sharing | `sharing` | Foundation | PARTIALLY_IMPLEMENTED |

Individual specifications: [PACKAGES/](PACKAGES/README.md).

## Dedup decisions (why these 33)

| Pair | Decision |
|---|---|
| Identity vs Security | Both exist. Identity owns AuthN/AuthZ. Security owns KMS / posture / WAF catalog. |
| Configuration vs Metadata | Settings values vs dictionary / UI / validation. |
| Event bus vs Messaging vs Notification | Facts vs jobs vs human/channel delivery. |
| Audit vs Logging vs Monitoring | Immutable trail vs technical logs vs metrics / SLO. |
| Reporting vs Dashboard | Datasets / exports vs boards / widgets. |
| Feature vs Licensing | Rollout vs commercial entitlement. |
| API vs Integration | Inbound API product vs outbound connectors. |
| File / Media vs Document vs Output | Bytes vs document identity vs determination / render. |

## Reserved

- Do not invent `p34+` in this phase.
- Units of measure and FX remain in `p02_organization`.
- Payment, tax, and accounting are business / finance concerns, not new platform numbers.
