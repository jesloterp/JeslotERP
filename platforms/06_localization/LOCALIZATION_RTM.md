# JeslotERP Localization Platform — Requirements Traceability Matrix (RTM)

**Version:** 1.0  
**Last updated:** 2026-09-12  
**Scope:** 100% coverage of LOCALIZATION_GUIDE.md + LOCALIZATION_SCHEMA.md + LOCALIZATION_API.md  
**Platform:** platforms.p06_localization  

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SCHEMA-T01 | LOCALIZATION_SCHEMA §2 | Table i18n_language — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T02 | LOCALIZATION_SCHEMA §2 | Table i18n_script — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T03 | LOCALIZATION_SCHEMA §2 | Table i18n_territory — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T04 | LOCALIZATION_SCHEMA §2 | Table i18n_locale — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T05 | LOCALIZATION_SCHEMA §2 | Table i18n_calendar — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T06 | LOCALIZATION_SCHEMA §2 | Table i18n_currency — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T07 | LOCALIZATION_SCHEMA §2 | Table i18n_timezone — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T08 | LOCALIZATION_SCHEMA §2 | Table i18n_locale_alias — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T09 | LOCALIZATION_SCHEMA §2 | Table i18n_fallback_rule — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T10 | LOCALIZATION_SCHEMA §2 | Table i18n_channel — Locale / CLDR catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T11 | LOCALIZATION_SCHEMA §2 | Table i18n_namespace — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T12 | LOCALIZATION_SCHEMA §2 | Table i18n_message_key — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T13 | LOCALIZATION_SCHEMA §2 | Table i18n_message — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T14 | LOCALIZATION_SCHEMA §2 | Table i18n_message_variant — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T15 | LOCALIZATION_SCHEMA §2 | Table i18n_message_argument — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T16 | LOCALIZATION_SCHEMA §2 | Table i18n_message_tag — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T17 | LOCALIZATION_SCHEMA §2 | Table i18n_message_key_tag — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T18 | LOCALIZATION_SCHEMA §2 | Table i18n_message_context — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T19 | LOCALIZATION_SCHEMA §2 | Table i18n_message_screenshot — Message catalog | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T20 | LOCALIZATION_SCHEMA §2 | Table i18n_format_profile — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T21 | LOCALIZATION_SCHEMA §2 | Table i18n_number_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T22 | LOCALIZATION_SCHEMA §2 | Table i18n_currency_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T23 | LOCALIZATION_SCHEMA §2 | Table i18n_date_time_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T24 | LOCALIZATION_SCHEMA §2 | Table i18n_address_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T25 | LOCALIZATION_SCHEMA §2 | Table i18n_person_name_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T26 | LOCALIZATION_SCHEMA §2 | Table i18n_phone_format — Regional formats | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T27 | LOCALIZATION_SCHEMA §2 | Table i18n_tenant_message_override — Overrides | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T28 | LOCALIZATION_SCHEMA §2 | Table i18n_company_message_override — Overrides | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T29 | LOCALIZATION_SCHEMA §2 | Table i18n_tenant_format_override — Overrides | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T30 | LOCALIZATION_SCHEMA §2 | Table i18n_user_locale_preference — Overrides | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T31 | LOCALIZATION_SCHEMA §2 | Table i18n_glossary — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T32 | LOCALIZATION_SCHEMA §2 | Table i18n_glossary_term — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T33 | LOCALIZATION_SCHEMA §2 | Table i18n_glossary_translation — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T34 | LOCALIZATION_SCHEMA §2 | Table i18n_tm_entry — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T35 | LOCALIZATION_SCHEMA §2 | Table i18n_tm_usage — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T36 | LOCALIZATION_SCHEMA §2 | Table i18n_term_violation — Glossary & TM | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T37 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_job — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T38 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_task — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T39 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_assignment — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T40 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_comment — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T41 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_review — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T42 | LOCALIZATION_SCHEMA §2 | Table i18n_translation_proposal — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T43 | LOCALIZATION_SCHEMA §2 | Table i18n_workflow_state_history — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T44 | LOCALIZATION_SCHEMA §2 | Table i18n_translator_profile — TMS workflow | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T45 | LOCALIZATION_SCHEMA §2 | Table i18n_package — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T46 | LOCALIZATION_SCHEMA §2 | Table i18n_package_item — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T47 | LOCALIZATION_SCHEMA §2 | Table i18n_changeset — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T48 | LOCALIZATION_SCHEMA §2 | Table i18n_changeset_item — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T49 | LOCALIZATION_SCHEMA §2 | Table i18n_approval — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T50 | LOCALIZATION_SCHEMA §2 | Table i18n_publish_version — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T51 | LOCALIZATION_SCHEMA §2 | Table i18n_publish_artifact — Packs & publish | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T52 | LOCALIZATION_SCHEMA §2 | Table i18n_coverage_report — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T53 | LOCALIZATION_SCHEMA §2 | Table i18n_coverage_finding — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T54 | LOCALIZATION_SCHEMA §2 | Table i18n_lint_rule — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T55 | LOCALIZATION_SCHEMA §2 | Table i18n_mt_provider — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T56 | LOCALIZATION_SCHEMA §2 | Table i18n_mt_run — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T57 | LOCALIZATION_SCHEMA §2 | Table i18n_mt_suggestion — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T58 | LOCALIZATION_SCHEMA §2 | Table i18n_pseudo_run — Quality / MT / pseudo | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T59 | LOCALIZATION_SCHEMA §2 | Table i18n_outbox — Plumbing & cache | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T60 | LOCALIZATION_SCHEMA §2 | Table i18n_idempotency_key — Plumbing & cache | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T61 | LOCALIZATION_SCHEMA §2 | Table i18n_resolve_cache — Plumbing & cache | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-T62 | LOCALIZATION_SCHEMA §2 | Table i18n_catalog_audit — Plumbing & cache | ORM models + persistence | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-RLS | LOCALIZATION_SCHEMA §13 RLS | FORCE RLS on tenant-scoped i18n tables | alembic `b0c1d2e3f4a5_enable_i18n_rls` + persistence/rls.py | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-MIXIN | LOCALIZATION_SCHEMA §1 Conventions | Enterprise ORM base columns | infrastructure/persistence/models/* | Implemented | test_all_62_i18n_tables_registered | Passed |
| SCHEMA-SEED | LOCALIZATION_SCHEMA §14 Seed minimum | Locale/channel/namespace/format/lint/permission seeds | alembic `a9b0c1d2e3f4` + catalog_store.seed_defaults | Implemented | test_fallback_chain_hi_in | Passed |
| SCHEMA-MIG | LOCALIZATION_SCHEMA DDL | Create i18n schema + 62 tables | alembic `a9b0c1d2e3f4_create_i18n_schema` | Implemented | test_all_62_i18n_tables_registered | Passed |
| GUIDE-WIRE | LOCALIZATION_GUIDE module load | Register LocalizationModule in API | apps/api/main.py load_modules + exception handlers | Implemented | test_load_modules_includes_localization | Passed |
| SCHEMA-ENUM | LOCALIZATION_SCHEMA §3 Enumerations | Direction/channel/origin enums | domain/enums.py | Implemented | test_origin_layers_order | Passed |
| GUIDE-01 | LOCALIZATION_GUIDE §1 Purpose | L10n control plane module | platforms/p06_localization | Implemented | test_module_plugin_contract | Passed |
| GUIDE-02 | LOCALIZATION_GUIDE §1 Owns | ICU messages, formats, TMS, packs | ORM + services + HTTP | Implemented | test_all_62_i18n_tables_registered | Passed |
| GUIDE-03 | LOCALIZATION_GUIDE §2 Architecture | Depends p01 + p03 | LocalizationModule.dependencies | Implemented | test_module_plugin_contract | Passed |
| GUIDE-04 | LOCALIZATION_GUIDE §3 Effective-first | Clients use /effective/* | effective.py | Implemented | test_effective_pack_success_and_etag | Passed |
| GUIDE-05 | LOCALIZATION_GUIDE §3 Layer provenance | origin_layer on debug resolve | resolve_messages debug | Implemented | test_debug_includes_origin_layer | Passed |
| GUIDE-06 | LOCALIZATION_GUIDE §3 ICU MessageFormat | Server format endpoint | icu_engine.format_message | Implemented | test_effective_format_icu_plural | Passed |
| GUIDE-07 | LOCALIZATION_GUIDE §3 Fallback chain | Deterministic locale fallback | fallback_chain.py | Implemented | test_fallback_chain_hi_in | Passed |
| GUIDE-08 | LOCALIZATION_GUIDE §3 RTL | direction for ar-* locales | resolve_messages + locales | Implemented | test_resolve_includes_rtl_direction_for_ar | Passed |
| GUIDE-09 | LOCALIZATION_GUIDE §5 Permissions | i18n.* catalog | permissions/catalog.py | Implemented | test_wildcard_permission_allows | Passed |
| GUIDE-10 | LOCALIZATION_GUIDE §6 Glossary gates | Lint on submit | glossary_lint + tms submit | Implemented | test_glossary_lint_violation_422 | Passed |
| GUIDE-11 | LOCALIZATION_GUIDE §7 Publish immutability | Checksum/version artifacts | publisher.publish_bundle | Implemented | test_publish_increments_bundle_version | Passed |
| GUIDE-12 | LOCALIZATION_GUIDE §7 Coverage gates | BLOCKER blocks publish | coverage + publisher | Implemented | test_publish_blocked_when_coverage_gate_fails | Passed |
| GUIDE-13 | LOCALIZATION_GUIDE §8 MT assist-only | MT creates reviewable proposals | mt.py accept-to-task | Implemented | test_pseudo_and_mt_runs | Passed |
| GUIDE-14 | LOCALIZATION_GUIDE §9 XLIFF import-export | Import/export endpoints | import_export.py | Implemented | test_import_export | Passed |
| GUIDE-15 | LOCALIZATION_GUIDE §15 DoD ICU tests | en/hi/ar plural tests | test_icu_engine.py | Implemented | test_ar_plural_few | Passed |
| API-06 | LOCALIZATION_API §6 Effective | GET /effective/pack ETag/304 | effective.py | Implemented | test_effective_pack_if_none_match_304 | Passed |
| API-06b | LOCALIZATION_API §6 Effective | POST /effective/messages | effective.py | Implemented | test_effective_messages_post | Passed |
| API-06c | LOCALIZATION_API §6 Effective | POST /effective/format ICU | effective.py | Implemented | test_effective_format_icu_plural | Passed |
| API-06d | LOCALIZATION_API §6 Effective | GET /effective/formats | effective.py | Implemented | test_effective_formats_and_locale_shell | Passed |
| API-06e | LOCALIZATION_API §6 Effective | GET /effective/locale shell | effective.py | Implemented | test_effective_formats_and_locale_shell | Passed |
| API-07 | LOCALIZATION_API §7 CLDR | Locales/languages/territories/fallback | locales.py | Implemented | test_fallback_rules_get_and_put | Passed |
| API-08 | LOCALIZATION_API §8 Catalog | Namespaces/keys/messages/ICU validate | catalog.py | Implemented | test_namespace_crud_flow | Passed |
| API-09 | LOCALIZATION_API §9 Overrides | Overrides + preferences | overrides.py | Implemented | test_overrides_and_preferences | Passed |
| API-10 | LOCALIZATION_API §10 TMS | Jobs/tasks/proposals/reviews | tms.py | Implemented | test_tms_job_and_task_flow | Passed |
| API-11 | LOCALIZATION_API §11 Glossary/TM | Glossary + TM endpoints | glossary_tm.py | Implemented | test_glossary_lint_violation_422 | Passed |
| API-12 | LOCALIZATION_API §12 Packs/Publish | Packages/changesets/publish/bundles | packages_publish.py | Implemented | test_publish_and_versions | Passed |
| API-13 | LOCALIZATION_API §13 Quality | Coverage/lint/pseudo | quality.py | Implemented | test_coverage_scan_and_findings | Passed |
| API-14 | LOCALIZATION_API §14 MT | MT providers/runs/suggestions | mt.py | Implemented | test_pseudo_and_mt_runs | Passed |
| API-15 | LOCALIZATION_API §15 Import/Export | XLIFF/JSON export/import | import_export.py | Implemented | test_import_export | Passed |
| API-16 | LOCALIZATION_API §16 Audit | GET /audit | audit.py | Implemented | test_audit_requires_permission | Passed |
| API-17 | LOCALIZATION_API §17 Internal | hydrate/cache/health internal | internal.py | Implemented | test_internal_hydrate_labels | Passed |
| API-LOOKUP | LOCALIZATION_API §7 channels | GET /lookups/channels | lookups.py | Implemented | test_openapi_needles_present | Passed |
| API-HEALTH | LOCALIZATION_API §17 health | GET /health/live + /version | health.py | Implemented | test_health_live | Passed |
| API-PERM | LOCALIZATION_API §5 Permissions | i18n.* gates | dependencies/permissions.py | Implemented | test_missing_permission_raises_403 | Passed |
| API-ERR | LOCALIZATION_API §4 Errors | I18nError envelope | exception_handlers.py | Implemented | test_package_install_checksum_failure | Passed |
| I18N-SOR-08 | TASK-SOR-008 | Durable messages / ICU overlays on Postgres | catalog_repository + override_repository | Implemented | `test_durable_sor` | PASS |

**Total RTM rows:** 103
