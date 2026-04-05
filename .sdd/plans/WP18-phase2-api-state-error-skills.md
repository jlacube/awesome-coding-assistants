---
lane: to_do
review_status: has_feedback
---

# WP18 - Phase 2: API Contracts + State Machines + Error Catalogs Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/003-planner-v2.spec.md` |
| Priority | P1 |
| Lane | for_review |
| Depends on | WP14, WP15 |
| Goal | Implement the plan-api-contracts, plan-state-machines, and plan-error-catalogs skills that generate API endpoint definitions, state transition validators, and error code constants per WP |
| Status | Complete |
| Independent Test | Dispatch all 3 skills against a plan accumulator. Verify: applicable WPs have `api-contracts.<ext>`, `state-machines.<ext>`, and `error-catalog.<ext>` in their contracts directories; all files have manifest headers; error codes include HTTP status mappings |
| Parallelisable | Yes (with WP16, WP17, WP19 after WP14+WP15 complete) |
| Prompt | `.sdd/plans/WP18-phase2-api-state-error-skills.md` |

## Objective

Implement three Phase 2 contract generation skills: `plan-api-contracts` (request/response types, endpoint paths, auth requirements, error responses), `plan-state-machines` (state enums, transition validators, side effects), and `plan-error-catalogs` (error code constants, HTTP mappings, message templates). These skills read the plan accumulator and generate language-specific contract files scoped per WP.

## Spec References

FR-043 through FR-050, Section 4.7, Section 4.8, Section 4.9, Section 7.2

## Tasks

### T18-01 - Implement plan-api-contracts SKILL.md structure

- **Description**: Replace the stub `plan-api-contracts/SKILL.md` with the full skill implementation. Define purpose, input contract (Phase 2, 9 inputs), execution sequence, and output format.
- **Spec refs**: FR-043, FR-023, FR-024
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md follows the common plan-skill contract with phase=2 (FR-023)
  - [x] Skill reads plan state before generating contracts (FR-024, FR-016)
  - [x] Skill references `.github/skills/PLAN-SKILL-CONTRACT.md` for the common contract
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - API contracts are generated only for WPs that introduce or modify API endpoints

### T18-02 - Implement API contract generation logic

- **Description**: Write instructions for generating `api-contracts.<ext>` files per WP: request type definitions, response type definitions, endpoint path constants, and auth requirement declarations.
- **Spec refs**: FR-043, FR-044
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL generate request type definitions with all fields typed and validated (FR-043.1)
  - [x] The skill SHALL generate response type definitions with all fields typed (FR-043.2)
  - [x] The skill SHALL include endpoint path constants (FR-043.3)
  - [x] The skill SHALL include auth requirement declarations per endpoint (FR-043.4)
  - [x] API contracts SHALL match the spec's companion artifact `api-contracts.<ext>` (FR-044)
  - [x] Output file: `.sdd/plans/contracts/<WP-slug>/api-contracts.<ext>` (FR-043)
- **Test requirements**: BDD (US-02 Scenario 1)
- **Depends on**: T18-01
- **Implementation Guidance**:
  - Auth declarations: annotate each endpoint with its auth mechanism (e.g., Bearer token, API key, none)
  - Path constants: define as string constants (e.g., `const GET_USERS = '/api/v1/users'`)
  - If spec artifact does not exist, generate from spec prose

### T18-03 - Implement error response types per endpoint

- **Description**: Write instructions to include all applicable error response types per endpoint: 400, 401, 403, 404, 409, 422, 500 with typed error response schemas.
- **Spec refs**: FR-045
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each endpoint contract SHALL include all applicable error response types (FR-045)
  - [x] Error responses SHALL cover: 400, 401, 403, 404, 409, 422, 500 (FR-045)
  - [x] Each error response SHALL have a typed error response schema (FR-045)
  - [x] Not all error codes apply to every endpoint -- only include applicable ones
- **Test requirements**: BDD (US-02 Scenario 1)
- **Depends on**: T18-02
- **Implementation Guidance**:
  - A GET endpoint typically needs 401, 403, 404, 500
  - A POST endpoint typically needs 400, 401, 403, 409, 422, 500
  - Error response schemas should reference the error catalog types from plan-error-catalogs

### T18-04 - Implement plan-state-machines SKILL.md

- **Description**: Replace the stub `plan-state-machines/SKILL.md` with the full skill implementation. Write instructions for generating `state-machines.<ext>` files per WP that creates or modifies stateful entities.
- **Spec refs**: FR-046, FR-047, FR-023, FR-024
- **Parallel**: Yes (with T18-01 through T18-03)
- **Acceptance criteria**:
  - [x] SKILL.md follows the common plan-skill contract with phase=2 (FR-023)
  - [x] The skill SHALL generate state enum definitions with all valid values (FR-046.1)
  - [x] The skill SHALL generate transition validation functions (from-state, to-state, guard) (FR-046.2)
  - [x] The skill SHALL include side effect declarations per transition (FR-046.3)
  - [x] State machines SHALL match the spec's companion artifact `state-machines.<ext>` (FR-047)
  - [x] Output file: `.sdd/plans/contracts/<WP-slug>/state-machines.<ext>` (FR-046)
  - [x] No state machine files generated for WPs without stateful entities
- **Test requirements**: BDD (US-02 Scenario 1, Scenario 2)
- **Depends on**: none
- **Implementation Guidance**:
  - State machines are only relevant for entities with a status/state field
  - Guard functions: conditions that must be true for a transition to be valid
  - Side effects: actions triggered by a transition (e.g., send email on status change to "approved")
  - If spec artifact does not exist, derive state machines from spec prose data model

### T18-05 - Implement plan-error-catalogs SKILL.md

- **Description**: Replace the stub `plan-error-catalogs/SKILL.md` with the full skill implementation. Write instructions for generating `error-catalog.<ext>` files per WP with error code constants, HTTP status mappings, and message templates.
- **Spec refs**: FR-048, FR-049, FR-050, FR-023, FR-024
- **Parallel**: Yes (with T18-01 through T18-04)
- **Acceptance criteria**:
  - [x] SKILL.md follows the common plan-skill contract with phase=2 (FR-023)
  - [x] The skill SHALL generate error code constants/enums (FR-048.1)
  - [x] The skill SHALL include HTTP status code mappings if applicable (FR-048.2)
  - [x] The skill SHALL include user-facing error message templates (FR-048.3)
  - [x] The skill SHALL include internal log message templates (FR-048.4)
  - [x] Error catalogs SHALL match the spec's companion artifact `error-catalog.<ext>` (FR-049)
  - [x] If an error code is shared across multiple WPs, the first WP SHALL define it fully; subsequent WPs SHALL import/reference (FR-050)
  - [x] Output file: `.sdd/plans/contracts/<WP-slug>/error-catalog.<ext>` (FR-048)
- **Test requirements**: BDD (US-02 Scenario 1, Scenario 3)
- **Depends on**: none
- **Implementation Guidance**:
  - Error code format: use string constants or enums (e.g., `AUTH_INVALID_TOKEN = 'AUTH_001'`)
  - Message templates may include placeholders (e.g., `"User {userId} not found"`)
  - Shared error deduplication follows the same pattern as data schema deduplication (FR-042): first WP by number defines, others import

### T18-06 - Implement manifest headers and 800-line compliance

- **Description**: Ensure all three skills include manifest headers per FR-039 and enforce the 800-line block limit per WP per FR-013 in their generation instructions.
- **Spec refs**: FR-039, FR-013
- **Parallel**: No
- **Acceptance criteria**:
  - [x] All three skills include manifest header generation instructions matching FR-039 format
  - [x] Each `Generated by` field correctly identifies the producing skill
  - [x] All three skills include instructions to split at 800 lines per WP if needed (FR-013)
- **Test requirements**: none
- **Depends on**: T18-02, T18-04, T18-05
- **Implementation Guidance**:
  - Manifest header `Generated by` values: `plan-api-contracts skill`, `plan-state-machines skill`, `plan-error-catalogs skill`

### T18-07 - Verify encoding compliance

- **Description**: Run automated Unicode check on all three skill files. Fix any violations.
- **Spec refs**: Section 9.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Zero prohibited Unicode characters in all three SKILL.md files
  - [x] All hyphens are ASCII `-`, all quotes are straight
  - [x] Python validation script confirms clean results
- **Test requirements**: unit (encoding validation)
- **Depends on**: T18-06
- **Implementation Guidance**:
  - Reuse the Python encoding check from WP14

## Implementation Notes

- These three skills are grouped because they share similar patterns: read plan accumulator, match against spec artifacts, generate per-WP contract files with deduplication rules
- All three produce Phase 2 output to `.sdd/plans/contracts/<WP-slug>/`
- Error catalogs and API contracts are closely related -- error response schemas in API contracts reference error catalog types

## Parallel Opportunities

- T18-01-T18-03 (API contracts), T18-04 (state machines), and T18-05 (error catalogs) can be worked simultaneously since they are independent skills
- WP18 as a whole can be parallelized with WP17 and WP19

## Risks & Mitigations

- **Risk**: Error catalogs and API contracts may define inconsistent error types. **Mitigation**: plan-cross-wp-validation (WP19) detects and resolves such inconsistencies.
- **Risk**: State machine generation may be N/A for specs without stateful entities. **Mitigation**: The skill produces no output for WPs without entities having status fields, as specified in the edge cases.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T14:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T14:30:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T18:00:00Z - review-coordinator - lane=to_do - Verdict: Changes Required (1 FAIL) -- awaiting remediation

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T18:00:00Z
> **Verdict**: Changes Required
> **Skills dispatched**: review-spec (FAIL)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked
- [PASS] Activity Log: Consistent lane transitions
- [WARN] Commit granularity: Single bulk commit (efcfb9a) for all 7 tasks
- [PASS] Encoding: No violations found

### Review Feedback

> Implementers: address every FB-XX item before returning for re-review.

- [ ] **FB-01**: [spec-adherence] FR-050 shared/ directory deviation - plan-error-catalogs introduces `shared/error-catalog.<ext>` for error codes used by 3+ WPs. FR-050 requires the first WP to define it fully and subsequent WPs to import from the first WP. Section 7.2 defines shared/ as containing only config-schema.<ext>. Remove the move-to-shared logic.
  File: .github/skills/plan-error-catalogs/SKILL.md. Expected: All deduplication follows first-WP-defines pattern. No error-catalog files written to shared/.
  Source skills: review-spec (SPEC-008)

### Warnings
- [WARN] PROC-003: Single bulk commit for all tasks.

### Cross-Correlation Notes
- Systemic pattern: shared/ directory deduplication deviation also appears in WP17 (plan-data-schemas, plan-interface-contracts). Same root cause across 3 skills.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 16 | 0 | 1 |
| **Total** | **19** | **1** | **1** |
