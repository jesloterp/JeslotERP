# Platform deep specifications

This folder holds the **accepted deeper specifications** for the thirty-three
kernel packages. Chetan Patel added these documents so serious developers can
see GUIDE / SCHEMA / API / RTM / implementation-record depth — not only the
public summary pages.

## How to read this folder versus `PACKAGES/`

| Location | Audience | Depth |
|---|---|---|
| [`../PACKAGES/`](../PACKAGES/README.md) | Architects and new contributors | Public-safe purpose, status, dependencies |
| `platforms/NN_<name>/` | Implementers | Guide, conceptual schema, API groups, RTM, records |

Canonical package ids remain `p01_identity` … `p33_sharing`. Folder numbers
here (`01_identity`) match that numbering. Do not invent `p34+`.

## Status honesty

These records describe a **SoR-Live kernel**, not a Production product and not
an already-open source tree. Provider ports may be fail-closed. Threat-model
files are **unsigned** artifacts. They are not a security sign-off.

## Public / private rule

These documents must stay free of credentials, private hosts, customer data,
and unpublished product brands. If a later edit reintroduces those, reject it.

GUIDE “class of products” lists name well-known enterprise systems as
**reference patterns** only. They are not compatibility, certification, or
affiliation claims. See [../TRADEMARKS.md](../TRADEMARKS.md).

See [`../DEVELOPMENT/PUBLICATION_GUIDE.md`](../DEVELOPMENT/PUBLICATION_GUIDE.md).

## Index

| # | Folder | Package | Public summary |
|---|---|---|---|
| 01 | `01_identity` | `p01_identity` | [P01](../PACKAGES/P01_IDENTITY.md) |
| 02 | `02_organization` | `p02_organization` | [P02](../PACKAGES/P02_ORGANIZATION.md) |
| 03 | `03_configuration` | `p03_configuration` | [P03](../PACKAGES/P03_CONFIGURATION.md) |
| 04 | `04_business_partner` | `p04_business_partner` | [P04](../PACKAGES/P04_BUSINESS_PARTNER.md) |
| 05 | `05_metadata` | `p05_metadata` | [P05](../PACKAGES/P05_METADATA.md) |
| 06 | `06_localization` | `p06_localization` | [P06](../PACKAGES/P06_LOCALIZATION.md) |
| 07 | `07_number_series` | `p07_number_series` | [P07](../PACKAGES/P07_NUMBER_SERIES.md) |
| 08 | `08_file_media` | `p08_file_media` | [P08](../PACKAGES/P08_FILE_MEDIA.md) |
| 09 | `09_document` | `p09_document` | [P09](../PACKAGES/P09_DOCUMENT.md) |
| 10 | `10_process` | `p10_process` | [P10](../PACKAGES/P10_PROCESS.md) |
| 11 | `11_rules` | `p11_rules` | [P11](../PACKAGES/P11_RULES.md) |
| 12 | `12_feature` | `p12_feature` | [P12](../PACKAGES/P12_FEATURE.md) |
| 13 | `13_event_bus` | `p13_event_bus` | [P13](../PACKAGES/P13_EVENT_BUS.md) |
| 14 | `14_messaging` | `p14_messaging` | [P14](../PACKAGES/P14_MESSAGING.md) |
| 15 | `15_notification` | `p15_notification` | [P15](../PACKAGES/P15_NOTIFICATION.md) |
| 16 | `16_cache` | `p16_cache` | [P16](../PACKAGES/P16_CACHE.md) |
| 17 | `17_scheduler` | `p17_scheduler` | [P17](../PACKAGES/P17_SCHEDULER.md) |
| 18 | `18_search` | `p18_search` | [P18](../PACKAGES/P18_SEARCH.md) |
| 19 | `19_audit` | `p19_audit` | [P19](../PACKAGES/P19_AUDIT.md) |
| 20 | `20_logging` | `p20_logging` | [P20](../PACKAGES/P20_LOGGING.md) |
| 21 | `21_monitoring` | `p21_monitoring` | [P21](../PACKAGES/P21_MONITORING.md) |
| 22 | `22_api` | `p22_api` | [P22](../PACKAGES/P22_API.md) |
| 23 | `23_integration` | `p23_integration` | [P23](../PACKAGES/P23_INTEGRATION.md) |
| 24 | `24_reporting` | `p24_reporting` | [P24](../PACKAGES/P24_REPORTING.md) |
| 25 | `25_dashboard` | `p25_dashboard` | [P25](../PACKAGES/P25_DASHBOARD.md) |
| 26 | `26_licensing` | `p26_licensing` | [P26](../PACKAGES/P26_LICENSING.md) |
| 27 | `27_ai` | `p27_ai` | [P27](../PACKAGES/P27_AI.md) |
| 28 | `28_security` | `p28_security` | [P28](../PACKAGES/P28_SECURITY.md) |
| 29 | `29_extensibility` | `p29_extensibility` | [P29](../PACKAGES/P29_EXTENSIBILITY.md) |
| 30 | `30_alm` | `p30_alm` | [P30](../PACKAGES/P30_ALM.md) |
| 31 | `31_privacy` | `p31_privacy` | [P31](../PACKAGES/P31_PRIVACY.md) |
| 32 | `32_output` | `p32_output` | [P32](../PACKAGES/P32_OUTPUT.md) |
| 33 | `33_sharing` | `p33_sharing` | [P33](../PACKAGES/P33_SHARING.md) |

Typical files per folder (not every package has every file):

- `*_GUIDE.md` — developer integration
- `*_SCHEMA.md` — conceptual table map
- `*_API.md` — resource groups and contracts
- `*_RTM.md` — requirements traceability
- `*_IMPLEMENTATION_RECORD.md` — shipped-slice evidence
