---
lane: done
---

# WP42 - Return Handoff Schemas & Shared Base

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP40 |
| Goal | Create 3 return handoff schemas and a shared base schema with reusable validation patterns |
| Status | Complete |
| Independent Test | List `.github/schemas/` and verify reviewer-to-orchestrator.schema.yaml, coder-complete-to-orchestrator.schema.yaml, docs-agent-to-orchestrator.schema.yaml, and base-handoff.schema.yaml all exist. Read each and verify required fields. |
| Parallelisable | Yes (with WP41, WP43) |
| Prompt | `.sdd/plans/WP42-return-schemas-base.md` |

## Objective

Create three return handoff schema files that formalize the completion signals from the Review Coordinator, Coder, and Docs Agent back to the Orchestrator. Also create a shared base schema at `.github/schemas/base-handoff.schema.yaml` containing reusable validation patterns (WP file existence, lane validation, path format) that individual schemas can reference. This completes the handoff schema coverage so every directional agent-to-agent path has an explicit contract.

## Spec References

FR-021, FR-022, FR-023, FR-024, FR-025, FR-026, FR-027, FR-049, FR-050, FR-051, Section 7.3 (Handoff Schema), Section 7.4 (Base Handoff Schema), Section 7.7 (Return Handoff Context), Section 8.3 (Handoff Schema Validation Interface), US-08, US-11

## Tasks

### T42-01 - Create base-handoff.schema.yaml

- **Description**: Create the shared base schema file at `.github/schemas/base-handoff.schema.yaml` defining three reusable validation patterns: wp_file_exists, lane_value_valid, and file_path_format.
- **Spec refs**: FR-049, FR-050, Section 7.4
- **Parallel**: No (dependent schemas reference this)
- **Acceptance criteria**:
  - [x] File exists at `.github/schemas/base-handoff.schema.yaml` (FR-049)
  - [x] Contains `schema: base/v1` identifier
  - [x] Defines `validation_patterns.wp_file_exists` with check and error fields (FR-050)
  - [x] Defines `validation_patterns.lane_value_valid` with check, enum_ref, and error fields (FR-050)
  - [x] Defines `validation_patterns.file_path_format` with check, pattern, and error fields (FR-050)
  - [x] `lane_value_valid` references enums.yaml for valid values
- **Test requirements**: content (YAML parse + pattern verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the Section 7.4 data model for exact field structure
  - Error E-024 (BASE_SCHEMA_NOT_FOUND): if this file is missing, individual schemas fall back to inline rules
  - The `file_path_format` pattern SHALL validate paths matching `^\.sdd/plans/WP\d{2}-[a-z0-9-]+\.md$` (Section 10.2.4)
  - Files to create: `.github/schemas/base-handoff.schema.yaml`

### T42-02 - Create reviewer-to-orchestrator.schema.yaml

- **Description**: Create the return handoff schema for the Review Coordinator's completion signal to the Orchestrator.
- **Spec refs**: FR-021, FR-022, FR-027, Section 7.7
- **Parallel**: Yes (with T42-03, T42-04)
- **Acceptance criteria**:
  - [x] File exists at `.github/schemas/reviewer-to-orchestrator.schema.yaml` (FR-021)
  - [x] Schema follows handoff/v1 format with all required sections (FR-027)
  - [x] `context_fields` includes `wp_path` (string, required) (FR-022)
  - [x] `context_fields` includes `verdict` (enum: approved, changes_required; required) (FR-022)
  - [x] `context_fields` includes `updated_lane` (enum from lane values; required) (FR-022)
  - [x] `source_agent` is "5. Reviewer" and `target_agent` is "1. Orchestrator"
  - [x] Contains `base_schema` reference to base-handoff.schema.yaml (FR-051)
- **Test requirements**: content (YAML parse), BDD (US-08 Scenario 1, Scenario 2)
- **Depends on**: T42-01
- **Implementation Guidance**:
  - Follow existing handoff schema format from `.github/schemas/spec-to-planner.schema.yaml` as template
  - `validation_rules` SHALL verify `updated_lane` matches `verdict`: approved->done, changes_required->to_do
  - Error E-021: missing required context field -> Orchestrator halts
  - Error E-022: invalid field value -> Orchestrator halts with expected vs actual
  - Files to create: `.github/schemas/reviewer-to-orchestrator.schema.yaml`

### T42-03 - Create coder-complete-to-orchestrator.schema.yaml

- **Description**: Create the return handoff schema for the Coder's completion signal to the Orchestrator.
- **Spec refs**: FR-023, FR-024, FR-027, Section 7.7
- **Parallel**: Yes (with T42-02, T42-04)
- **Acceptance criteria**:
  - [x] File exists at `.github/schemas/coder-complete-to-orchestrator.schema.yaml` (FR-023)
  - [x] Schema follows handoff/v1 format (FR-027)
  - [x] `context_fields` includes `wp_path` (string, required) (FR-024)
  - [x] `context_fields` includes `lane_confirmation` (literal value `for_review`, required) (FR-024)
  - [x] `source_agent` is "4. Coder" and `target_agent` is "1. Orchestrator"
  - [x] Contains `base_schema` reference (FR-051)
- **Test requirements**: content (YAML parse)
- **Depends on**: T42-01
- **Implementation Guidance**:
  - `validation_rules` SHALL verify `lane_confirmation` is literally `for_review`
  - Error E-022: if `lane_confirmation` is not `for_review`, Orchestrator halts
  - Files to create: `.github/schemas/coder-complete-to-orchestrator.schema.yaml`

### T42-04 - Create docs-agent-to-orchestrator.schema.yaml

- **Description**: Create the return handoff schema for the Docs Agent's completion signal to the Orchestrator.
- **Spec refs**: FR-025, FR-026, FR-027, Section 7.7
- **Parallel**: Yes (with T42-02, T42-03)
- **Acceptance criteria**:
  - [x] File exists at `.github/schemas/docs-agent-to-orchestrator.schema.yaml` (FR-025)
  - [x] Schema follows handoff/v1 format (FR-027)
  - [x] `context_fields` includes `wp_path` (string, required) (FR-026)
  - [x] `context_fields` includes `docs_completed` (boolean, required) (FR-026)
  - [x] `source_agent` is "6. Docs Agent" and `target_agent` is "1. Orchestrator"
  - [x] Contains `base_schema` reference (FR-051)
- **Test requirements**: content (YAML parse), BDD (US-08 Scenario 3)
- **Depends on**: T42-01
- **Implementation Guidance**:
  - `validation_rules` SHALL verify `docs_completed` is true (false is not a valid completion signal)
  - If `docs_completed` is not true, Orchestrator logs a warning and treats docs as incomplete
  - Files to create: `.github/schemas/docs-agent-to-orchestrator.schema.yaml`

### T42-05 - Link existing schemas to base schema

- **Description**: Update at least 2 existing handoff schema files to include a `base_schema` field referencing `.github/schemas/base-handoff.schema.yaml` and replace duplicated validation rules with `$ref`-style references.
- **Spec refs**: FR-051, Section 4.10
- **Parallel**: No (modifies existing files)
- **Acceptance criteria**:
  - [x] At least 2 existing schemas contain `base_schema: .github/schemas/base-handoff.schema.yaml` (FR-051)
  - [x] Duplicated validation rules (WP file existence, lane validation) are replaced with base schema references
  - [x] If base schema cannot be loaded, individual schemas' inline rules are used as fallback (FR-049)
- **Test requirements**: content (YAML parse), integration (cross-file reference check)
- **Depends on**: T42-01
- **Implementation Guidance**:
  - Good candidates: `spec-to-planner.schema.yaml` and `coder-to-reviewer.schema.yaml` (both validate WP-related artifacts)
  - Add `base_schema` field after the `description` field
  - Keep original inline rules as fallback comments
  - Files to modify: at least 2 files in `.github/schemas/`

### T42-06 - Verify schema structural consistency

- **Description**: Verify all 3 new return schemas and the base schema follow the handoff/v1 structure. Confirm cross-file references resolve correctly.
- **Spec refs**: FR-027, Section 11.3 (integration tests)
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [x] All 3 return schemas have: schema, source_agent, target_agent, description, required_artifacts, required_state, context_fields, validation_rules sections (FR-027)
  - [x] Base schema references in all schemas resolve to the correct file path
  - [x] Enum values in schema validation rules match enums.yaml registry values
- **Test requirements**: integration (cross-file consistency)
- **Depends on**: T42-02, T42-03, T42-04, T42-05
- **Implementation Guidance**:
  - Cross-reference Section 11.3 integration test table for verification criteria
  - Check that agent names match actual agent file names

## Implementation Notes

- All deliverables are YAML schema files -- no executable code
- Follow the existing handoff/v1 format for structural consistency (Design Decision 3, Section 9.4)
- The base schema is optional -- individual schemas SHALL have inline fallback rules (FR-049)
- Total schema file count after this WP: 11 existing + 3 new return + 1 base = 15 files in `.github/schemas/`
- Return handoff schemas are lightweight (fewer required_artifacts than forward schemas) but use the same format

## Risks & Mitigations

- **Risk**: Existing schemas may have inconsistent structures. **Mitigation**: Use spec-to-planner.schema.yaml as the reference template -- it was validated as the example in the spec.
- **Risk**: Base schema references may break if the file is moved. **Mitigation**: Use relative paths and document the expected location.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-07T00:10:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (WARN), review-security (N/A), review-quality (PASS), review-tests (N/A), review-architecture (PASS), review-performance (N/A), review-docs (PASS), review-deps (N/A)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked across T42-01 through T42-06
- [PASS] Activity Log: Consistent lane transitions (planned -> doing -> for_review)
- [PASS] Commit granularity: 3 implementation commits (T42-01, T42-02/03/04, T42-05) plus plan submission
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. No FB-XX items to address.

### Warnings
- [WARN] Agent name inconsistency: `reviewer-to-orchestrator.schema.yaml` uses `source_agent: "5. Reviewer"` while all other schemas use `"5. Review Coordinator"`. The WP plan's T42-02 acceptance criteria specified this value, so the Coder followed the plan correctly. Consider correcting to "5. Review Coordinator" for consistency. (review-spec SPEC-014)

### Cross-Correlation Notes
- No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 13 | 1 | 0 |
| review-security | 0 | 0 | 0 |
| review-quality | 4 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 4 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 1 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **26** | **1** | **0** |

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-07T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-07T00:01:00Z - coder - T42-01 completed - Created base-handoff.schema.yaml
- 2026-04-07T00:02:00Z - coder - T42-02 completed - Created reviewer-to-orchestrator.schema.yaml
- 2026-04-07T00:02:01Z - coder - T42-03 completed - Created coder-complete-to-orchestrator.schema.yaml
- 2026-04-07T00:02:02Z - coder - T42-04 completed - Created docs-agent-to-orchestrator.schema.yaml
- 2026-04-07T00:03:00Z - coder - T42-05 completed - Linked spec-to-planner and coder-to-reviewer schemas to base
- 2026-04-07T00:04:00Z - coder - T42-06 completed - Verified structural consistency across all schemas
- 2026-04-07T00:05:00Z - coder - lane=for_review - All tasks complete, all acceptance criteria met
- 2026-04-07T00:10:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARNs)
