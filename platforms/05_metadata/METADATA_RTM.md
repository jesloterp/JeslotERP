# JeslotERP Metadata Platform — Requirements Traceability Matrix (RTM)

**Version:** 1.0  
**Last updated:** 2026-09-10  
**Scope:** 100% coverage of METADATA_GUIDE.md + METADATA_SCHEMA.md + METADATA_API.md  
**Platform:** `platforms.p05_metadata`  

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SCHEMA-T01 | METADATA_SCHEMA §2 | Table `metadata_data_type` — Physical types lookup | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T02 | METADATA_SCHEMA §2 | Table `metadata_semantic_type` — Business semantic types | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T03 | METADATA_SCHEMA §2 | Table `metadata_ui_control` — UI controls | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T04 | METADATA_SCHEMA §2 | Table `metadata_validation_type` — Validation rule kinds | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T05 | METADATA_SCHEMA §2 | Table `metadata_relation_kind` — Relation cardinalities | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T06 | METADATA_SCHEMA §2 | Table `metadata_storage_strategy` — Persistence strategies | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T07 | METADATA_SCHEMA §2 | Table `metadata_channel` — Channel codes | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T08 | METADATA_SCHEMA §2 | Table `metadata_classification` — Data classification | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T09 | METADATA_SCHEMA §2 | Table `metadata_module` — Module namespace | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T10 | METADATA_SCHEMA §2 | Table `metadata_business_domain` — Semantic domain | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T11 | METADATA_SCHEMA §2 | Table `metadata_entity` — Entity descriptors | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T12 | METADATA_SCHEMA §2 | Table `metadata_field` — Field descriptors | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T13 | METADATA_SCHEMA §2 | Table `metadata_field_option` — Enum options | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T14 | METADATA_SCHEMA §2 | Table `metadata_relation` — Entity relations | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T15 | METADATA_SCHEMA §2 | Table `metadata_entity_index` — Index documentation | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T16 | METADATA_SCHEMA §2 | Table `metadata_entity_constraint` — Constraint documentation | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T17 | METADATA_SCHEMA §2 | Table `metadata_composite_field` — Composite sub-structure | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T18 | METADATA_SCHEMA §2 | Table `metadata_measure` — Report measures | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T19 | METADATA_SCHEMA §2 | Table `metadata_dimension` — Report dimensions | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T20 | METADATA_SCHEMA §2 | Table `metadata_metric_binding` — Measure bindings | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T21 | METADATA_SCHEMA §2 | Table `metadata_search_analyzer` — Search analyzer hints | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T22 | METADATA_SCHEMA §2 | Table `metadata_validation_rule` — Declarative validations | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T23 | METADATA_SCHEMA §2 | Table `metadata_expression` — Named safe ASTs | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T24 | METADATA_SCHEMA §2 | Table `metadata_field_default` — Default AST/value | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T25 | METADATA_SCHEMA §2 | Table `metadata_computed_field` — Virtual compute AST | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T26 | METADATA_SCHEMA §2 | Table `metadata_field_security` — FLS / mask / perms | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T27 | METADATA_SCHEMA §2 | Table `metadata_entity_security` — Entity default perms | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T28 | METADATA_SCHEMA §2 | Table `metadata_masking_policy` — Named mask strategies | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T29 | METADATA_SCHEMA §2 | Table `metadata_form` — Forms | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T30 | METADATA_SCHEMA §2 | Table `metadata_form_section` — Form sections | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T31 | METADATA_SCHEMA §2 | Table `metadata_form_field` — Form field placements | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T32 | METADATA_SCHEMA §2 | Table `metadata_form_variant` — Form channel/role variants | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T33 | METADATA_SCHEMA §2 | Table `metadata_list_view` — List views | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T34 | METADATA_SCHEMA §2 | Table `metadata_list_column` — List columns | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T35 | METADATA_SCHEMA §2 | Table `metadata_list_variant` — List variants | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T36 | METADATA_SCHEMA §2 | Table `metadata_filter` — Filters | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T37 | METADATA_SCHEMA §2 | Table `metadata_filter_field` — Filter fields | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T38 | METADATA_SCHEMA §2 | Table `metadata_action` — Actions | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T39 | METADATA_SCHEMA §2 | Table `metadata_action_variant` — Action variants | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T40 | METADATA_SCHEMA §2 | Table `metadata_inspector` — Inspector layouts | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T41 | METADATA_SCHEMA §2 | Table `metadata_entity_overlay` — Tenant overlays | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T42 | METADATA_SCHEMA §2 | Table `metadata_extension_value` — EAV extension values | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T43 | METADATA_SCHEMA §2 | Table `metadata_user_preference` — User list preferences | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T44 | METADATA_SCHEMA §2 | Table `metadata_package` — Installable packages | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T45 | METADATA_SCHEMA §2 | Table `metadata_package_item` — Package contents | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T46 | METADATA_SCHEMA §2 | Table `metadata_changeset` — Change batches | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T47 | METADATA_SCHEMA §2 | Table `metadata_changeset_item` — Changeset ops | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T48 | METADATA_SCHEMA §2 | Table `metadata_approval` — Approvals | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T49 | METADATA_SCHEMA §2 | Table `metadata_publish_version` — Publish versions | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T50 | METADATA_SCHEMA §2 | Table `metadata_publish_artifact` — Publish snapshots | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T51 | METADATA_SCHEMA §2 | Table `metadata_dependency_edge` — Dependency graph | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T52 | METADATA_SCHEMA §2 | Table `metadata_catalog_audit` — Catalog audit trail | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T53 | METADATA_SCHEMA §2 | Table `metadata_command_descriptor` — Command catalog | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T54 | METADATA_SCHEMA §2 | Table `metadata_query_descriptor` — Query catalog | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T55 | METADATA_SCHEMA §2 | Table `metadata_event_descriptor` — Event catalog | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T56 | METADATA_SCHEMA §2 | Table `metadata_drift_report` — Drift scan header | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T57 | METADATA_SCHEMA §2 | Table `metadata_drift_finding` — Drift findings | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T58 | METADATA_SCHEMA §2 | Table `metadata_outbox` — Transactional outbox | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T59 | METADATA_SCHEMA §2 | Table `metadata_idempotency_key` — Idempotency keys | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-T60 | METADATA_SCHEMA §2 | Table `metadata_resolve_cache` — Resolve cache entries | ORM + alembic e7f8a9b0c1d2 | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-RLS | §16 RLS | FORCE RLS on tenant-scoped metadata tables | alembic f8a9b0c1d2e3 + persistence/rls.py | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-MIXIN | §4 Mixins | Enterprise ORM base columns on metadata models | infrastructure/persistence/models/* | Implemented | test_all_60_metadata_tables_registered | Passed |
| SCHEMA-SEED | §17 Seed minimum | Lookup seeds via migration | e7f8a9b0c1d2 migration | Implemented | test_lookups_data_types_success | Passed |
| SCHEMA-ENUM | §3 Enumerations | Channel/classification/mask enums | domain/enums.py | Implemented | test_origin_layers_order | Passed |
| GUIDE-01 | §1 Purpose | Enterprise metadata control plane for dictionary, UI descriptors, effective resolution | platforms/p05_metadata module + services | Implemented | test_module_plugin_contract | Passed |
| GUIDE-02 | §1 Owns | Entity/field catalog, UI descriptors, semantic layer, FLS descriptors | ORM models + HTTP routers | Implemented | test_all_60_metadata_tables_registered | Passed |
| GUIDE-03 | §1 Does not own | Runtime setting values remain p03 | Boundary docs only | Documented | N/A (doc boundary) | N/A |
| GUIDE-04 | §2 Architecture | Depends p01–p03; integrates optional p06/p12 | MetadataModule.dependencies | Implemented | test_metadata_module_plugin | Passed |
| GUIDE-05 | §3 Effective-first | Production clients use /effective and /ui-pack | effective.py routers | Implemented | test_ui_pack_success_returns_entity_and_etag | Passed |
| GUIDE-06 | §4 Layer provenance | Every effective field includes origin_layer | effective_resolver.merge_layers | Implemented | test_tenant_layer_overrides_system_field | Passed |
| GUIDE-07 | §4 Resolve inputs | Channel, overlays, feature flags, publish_version | ResolveContext + query params | Implemented | test_ui_pack_if_none_match_returns_304 | Passed |
| GUIDE-08 | §5.1 Semantic types | Business semantic typing on fields | semantic.py + ORM semantic tables | Implemented | test_semantic_list_business_domains | Passed |
| GUIDE-09 | §5.2 Computed fields | Virtual fields via AST | expression_engine + validator | Implemented | test_valid_payload_with_allowed_custom_key | Passed |
| GUIDE-10 | §5.4 FLS | Field-level security + server redaction | security_fls.py + redaction.py | Implemented | test_redact_masks_row_values | Passed |
| GUIDE-11 | §5.5 Dependency graph | Impact before breaking changes | impact.py + ImpactGraph | Implemented | test_breaking_deprecate_blocked_by_default | Passed |
| GUIDE-12 | §5.6 Schema drift | Compare metadata to live DB | drift_scanner + drift routers | Implemented | test_drift_start_scan_returns_status | Passed |
| GUIDE-13 | §5.7 Packages | Installable industry packs | packages.py + package_installer | Implemented | test_packages_import_and_install_roundtrip | Passed |
| GUIDE-14 | §5.8 Publish governance | Immutable versions + rollback | catalog_store.PublishStore + publish_repository | Implemented | test_publish_versions_are_immutable_by_checksum | Passed |
| GUIDE-23 | §16 DoD HTTP SoR | Dictionary/overlay/publish on AsyncSession | dictionary/overlay/publish repositories | Implemented | test_persist_module_then_fetch_from_same_session | Passed |
| GUIDE-15 | §7 Permissions | metadata.* catalog/extension/publish codes | application/permissions/catalog.py | Implemented | test_permissions_catalog_from_guide | Passed |
| GUIDE-16 | §8 AST engine | Safe allow-list; forbid eval | expression_engine.py | Implemented | test_validate_ast_forbids_eval_node | Passed |
| GUIDE-17 | §9 Caching | ETag + Cache-Control on ui-pack | effective ui_pack handler | Implemented | test_ui_pack_success_returns_entity_and_etag | Passed |
| GUIDE-18 | §12 Domain events | Outbox for metadata changes | messaging/outbox | Implemented | test_module_plugin_contract | Passed |
| GUIDE-19 | §16 DoD effective resolve | Deterministic layered resolve | effective_resolver tests | Implemented | test_system_provenance_when_no_overlay | Passed |
| GUIDE-20 | §16 DoD AST tests | Reject forbidden nodes | expression tests | Implemented | test_ast_rejects_forbidden_op | Passed |
| GUIDE-21 | §16 DoD publish/impact | Breaking blocked without override | publish_gates + dictionary deprecate | Implemented | test_governance_publish_breaking_blocked_without_flag | Passed |
| GUIDE-22 | §16 DoD contract tests | Route surface + API contracts | test_route_surface + test_api_area_contracts | Implemented | test_all_api_needles_registered | Passed |
| API-5.1 | §5.1 UI pack | GET /ui-pack with ETag/304 | effective ui_pack_router | Implemented | test_ui_pack_if_none_match_returns_304 | Passed |
| API-5.2 | §5.2 Effective entity | GET /effective/entities/{entity_key} | effective.py | Implemented | test_effective_entity_success | Passed |
| API-5.3 | §5.3 Forms/lists/actions | GET effective form/list/actions | effective.py | Implemented | test_ui_authoring_get_form_stub | Passed |
| API-5.4 | §5.4 Debug trace | GET /effective/debug (manage) | effective.py | Implemented | test_effective_debug_requires_manage_permission | Passed |
| API-6 | §6 Validate | POST /validate with FLS strip | validate.py | Implemented | test_validate_success_strips_fls_denied_field | Passed |
| API-6-E | §6 Validate errors | GSTIN + custom key validation | validator.py | Implemented | test_validate_invalid_gstin_returns_errors | Passed |
| API-7 | §7 Dictionary | Modules/entities/fields CRUD | dictionary.py | Implemented | test_dictionary_list_modules_empty_ok | Passed |
| API-7-D | §7 Deprecate | Breaking deprecate → 409 | dictionary deprecate + ImpactGraph | Implemented | test_dictionary_deprecate_blocked_without_allow_breaking | Passed |
| API-8 | §8 Semantic | Domains/types/measures/dimensions | semantic.py | Implemented | test_semantic_entity_descriptor | Passed |
| API-9 | §9 Expressions | validate-ast + CRUD + evaluate | expressions.py | Implemented | test_expressions_validate_ast_safe | Passed |
| API-9-E | §9 AST forbid | Forbidden node error | expression_engine | Implemented | test_expressions_validate_ast_forbidden_node | Passed |
| API-10 | §10 FLS | Field/entity security + redact | security_fls.py | Implemented | test_fls_field_security_get_empty_default | Passed |
| API-10-R | §10 Redact | POST /redact masks rows | security_fls redact | Implemented | test_redact_masks_row_values | Passed |
| API-11 | §11 UI authoring | Forms/variants/inspectors/prefs | ui_authoring.py | Implemented | test_ui_authoring_put_form_persists_in_memory | Passed |
| API-12 | §12 Overlays | GET/PUT overlays extension perms | overlays.py | Implemented | test_overlays_put_denied_without_manage | Passed |
| API-13 | §13 Packages | import/install/export | packages.py | Implemented | test_packages_import_and_install_roundtrip | Passed |
| API-13-E | §13 Packages 404 | Missing package | packages.py | Implemented | test_packages_get_missing_returns_404 | Passed |
| API-14-C | §14 Changesets | Create/submit/apply workflow | governance.py | Implemented | test_governance_create_changeset | Passed |
| API-14-P | §14 Publish | Breaking publish gate | PublishStore | Implemented | test_governance_publish_breaking_blocked_without_flag | Passed |
| API-14-A | §14 Approvals | List/approve/reject | governance.py | Implemented | test_governance_approvals_list_requires_approve_permission | Passed |
| API-15 | §15 Impact | GET /impact + dependencies | impact.py | Implemented | test_impact_query_success_with_mock_db | Passed |
| API-15-D | §15 Impact auth | 403 without metadata.impact.read | permissions gate | Implemented | test_impact_denied_without_permission | Passed |
| API-16 | §16 Drift | POST/GET drift scans | drift.py | Implemented | test_drift_start_scan_returns_status | Passed |
| API-16-E | §16 Drift 404 | Unknown scan id | drift.py | Implemented | test_drift_get_unknown_scan_404 | Passed |
| API-17 | §17 Interop | commands/queries/events/contracts | interop.py | Implemented | test_interop_contracts_for_entity | Passed |
| API-18 | §18 Audit | GET /audit paginated | audit.py | Implemented | test_audit_list_empty_catalog | Passed |
| API-19 | §19 Lookups | data-types/channels/… | lookups.py | Implemented | test_lookups_data_types_success | Passed |
| API-19-D | §19 Lookups auth | 403 without catalog.read | permissions | Implemented | test_lookups_channels_denied_without_read | Passed |
| API-20 | §20 Internal mesh | Internal health/ui-pack/validate | internal.py | Implemented | test_internal_health_alive | Passed |
| API-20-V | §20 Internal validate | Requires tenant_id body fields | internal.py | Implemented | test_internal_validate_requires_tenant_body | Passed |
| API-4 | §4 Permissions | metadata.* wildcard + deny | permissions.py | Implemented | test_wildcard_metadata_star_allows_any_code | Passed |
| API-4-D | §4 Permissions deny | Missing permission 403 | permissions.py | Implemented | test_missing_permission_raises_403 | Passed |
| API-21 | §21 Caching | ETag header on ui-pack | effective.py | Implemented | test_ui_pack_success_returns_entity_and_etag | Passed |
| API-24-1 | §24 Checklist resolver | Layer precedence tests | test_effective_resolver | Implemented | test_tenant_layer_overrides_system_field | Passed |
| API-24-2 | §24 Checklist AST | Parse/eval/forbid suite | test_expression_engine | Implemented | test_validate_ast_forbids_eval_node | Passed |
| API-24-3 | §24 Checklist publish | Immutability + rollback | test_publisher | Implemented | test_rollback_creates_new_version_with_prior_payload | Passed |
| API-24-4 | §24 Checklist drift | Drift scanner unit tests | test_drift_scanner | Implemented | test_drift_mismatch_on_type | Passed |
| API-24-5 | §24 Checklist package | Checksum install gate | test_package_installer | Implemented | test_package_checksum_mismatch_detected | Passed |
| API-24-6 | §24 Checklist contracts | Breaking change HTTP 409 | test_api_area_contracts | Implemented | test_dictionary_deprecate_blocked_without_allow_breaking | Passed |
