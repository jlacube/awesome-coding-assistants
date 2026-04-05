---
skill: review-spec
wp: WP18-phase2-api-state-error-skills
spec: .sdd/specs/003-planner-v2.spec.md
reviewed_at: 2026-04-05T21:00:00Z
status: completed
review_round: 2
finding_counts:
  pass: 17
  warn: 0
  fail: 0
  na: 4
files_reviewed:
  - .sdd/specs/003-planner-v2.spec.md
  - .sdd/plans/WP18-phase2-api-state-error-skills.md
  - .github/skills/plan-api-contracts/SKILL.md
  - .github/skills/plan-state-machines/SKILL.md
  - .github/skills/plan-error-catalogs/SKILL.md
---

# review-spec Findings for WP18-phase2-api-state-error-skills

## Summary

**Re-review (round 2).** Evaluated 8 primary FRs (FR-043 through FR-050) and 4 supporting FRs (FR-013, FR-023, FR-024, FR-039) across all three skill implementations: plan-api-contracts, plan-state-machines, and plan-error-catalogs.

Previous round had 1 FAIL (SPEC-008: FR-050 deviation -- `shared/` directory for error codes used by 3+ WPs). The remediation removed the `shared/` directory logic entirely, replacing it with a correct first-WP-defines pattern and an explicit constraint prohibiting writes to `shared/`. SPEC-008 now passes.

No regressions detected in previously-passing items. No new issues introduced by the fix.

Overall: 17 PASS, 0 FAIL, 4 N/A.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-043
- **File**: .github/skills/plan-api-contracts/SKILL.md
- **Description**: The plan-api-contracts skill generates all 4 required components: (1) request type definitions with all fields typed and validated (Step 3c, FR-043.1), (2) response type definitions with all fields typed (Step 3d, FR-043.2), (3) endpoint path constants (Step 3b, FR-043.3), and (4) auth requirement declarations per endpoint (Step 3e, FR-043.4). Output path matches spec: `contracts/<WP-slug>/api-contracts.<ext>`. No regression from round 1.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-044
- **File**: .github/skills/plan-api-contracts/SKILL.md#L44-L55
- **Description**: Step 2 reads `spec_artifacts_dir` for companion artifact `api-contracts.<ext>`, treats it as canonical, and scopes per WP. When the artifact does not exist, the skill generates from spec prose and notes the derivation source in a comment. Both paths satisfy FR-044. No regression from round 1.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-045
- **File**: .github/skills/plan-api-contracts/SKILL.md#L132-L175
- **Description**: Step 4 defines all 7 required error response status codes (400, 401, 403, 404, 409, 422, 500) with typed error response schemas (`ApiErrorResponse` interface) and per-endpoint error mappings (`ENDPOINT_ERRORS`). Includes applicability rules that correctly scope error codes to endpoint types (e.g., 404 only for resource-by-ID endpoints, 409 only for creation endpoints with uniqueness constraints). No regression from round 1.

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-046
- **File**: .github/skills/plan-state-machines/SKILL.md
- **Description**: The plan-state-machines skill generates all 3 required components: (1) state enum definitions with all valid values (Step 3b, FR-046.1), (2) transition validation functions with from-state, to-state, and guard conditions (Step 3c, FR-046.2), and (3) side effect declarations per transition (Step 3d, FR-046.3). Output path matches spec: `contracts/<WP-slug>/state-machines.<ext>`. Correctly skips WPs without stateful entities (Step 1 skip rules, Constraints section). No regression from round 1.

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-047
- **File**: .github/skills/plan-state-machines/SKILL.md#L47-L59
- **Description**: Step 2 reads `spec_artifacts_dir` for companion artifact `state-machines.<ext>`, treats it as canonical, and scopes per WP. When the artifact does not exist, the skill derives state machines from the spec's data model section and notes the derivation source. Both paths satisfy FR-047. No regression from round 1.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-048
- **File**: .github/skills/plan-error-catalogs/SKILL.md
- **Description**: The plan-error-catalogs skill generates all 4 required components: (1) error code constants/enums (Step 4b, FR-048.1), (2) HTTP status code mappings with opt-out for non-HTTP apps (Step 4c, FR-048.2), (3) user-facing error message templates with placeholder support (Step 4d, FR-048.3), and (4) internal log message templates with diagnostic details (Step 4e, FR-048.4). Output path matches spec: `contracts/<WP-slug>/error-catalog.<ext>`. No regression from round 1.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-049
- **File**: .github/skills/plan-error-catalogs/SKILL.md#L45-L55
- **Description**: Step 2 reads `spec_artifacts_dir` for companion artifact `error-catalog.<ext>`, treats it as canonical, and scopes per WP. When the artifact does not exist, the skill extracts error codes from spec prose. Both paths satisfy FR-049. No regression from round 1.

### SPEC-008 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-050
- **File**: .github/skills/plan-error-catalogs/SKILL.md#L176-L195
- **Description**: **Previously FAIL (round 1).** The remediation replaced the `shared/` directory pattern with the correct first-WP-defines pattern. Step 5 now states: (1) the WP with the lowest number owns the full definition in its own contracts directory, (2) subsequent WPs import from the owner WP's contracts directory, (3) explicitly prohibits writing to `shared/` with rationale ("shared/ is reserved for config-schema files written by plan-cross-wp-validation only, Section 7.2"). The Constraints section also includes the explicit rule: "Only write to `<contracts_dir>/<WP-slug>/error-catalog.<ext>` -- do NOT write to `shared/`". This fully satisfies FR-050 and aligns with Section 7.2. Step 3 ownership resolution (first definer by WP number) remains correct and consistent with Step 5.

### SPEC-009 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-023
- **File**: .github/skills/plan-api-contracts/SKILL.md#L18-L30, .github/skills/plan-state-machines/SKILL.md#L18-L30, .github/skills/plan-error-catalogs/SKILL.md#L18-L30
- **Description**: All three skills define the 9-input contract table exactly matching FR-023: skill_path, plan_dir, contracts_dir, spec_path, spec_artifacts_dir, research_summary, target_language, patterns, phase. No regression from round 1.

### SPEC-010 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-024
- **File**: .github/skills/plan-api-contracts/SKILL.md#L34-L39, .github/skills/plan-state-machines/SKILL.md#L34-L39, .github/skills/plan-error-catalogs/SKILL.md#L34-L39
- **Description**: All three skills implement the 4-step execution sequence from FR-024: (1) read SKILL.md, (2) read plan state, (3) read spec + artifacts, (4) write contract files. No regression from round 1.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-039
- **File**: .github/skills/plan-api-contracts/SKILL.md#L63-L69, .github/skills/plan-state-machines/SKILL.md#L66-L72, .github/skills/plan-error-catalogs/SKILL.md#L93-L99
- **Description**: All three skills include manifest header templates matching FR-039 format with correctly identified `Generated by` values: `plan-api-contracts skill`, `plan-state-machines skill`, `plan-error-catalogs skill` respectively. No regression from round 1.

### SPEC-012 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-013
- **File**: .github/skills/plan-api-contracts/SKILL.md#L178-L180, .github/skills/plan-state-machines/SKILL.md#L159-L161, .github/skills/plan-error-catalogs/SKILL.md#L199-L201
- **Description**: All three skills include an 800-line block compliance section instructing to split generation if a WP's contract exceeds 800 lines, satisfying FR-013. No regression from round 1.

### SPEC-013 [PASS]
- **Checklist item**: Preconditions enforced / Postconditions produced
- **Requirement**: FR-043, FR-046, FR-048
- **File**: .github/skills/plan-api-contracts/SKILL.md#L43-L55, .github/skills/plan-state-machines/SKILL.md#L43-L59, .github/skills/plan-error-catalogs/SKILL.md#L43-L55
- **Description**: All three skills enforce preconditions by reading plan state and spec before generating output (Steps 1-2). All three produce postconditions by writing to the correct output paths: `contracts/<WP-slug>/api-contracts.<ext>`, `contracts/<WP-slug>/state-machines.<ext>`, `contracts/<WP-slug>/error-catalog.<ext>`. No regression from round 1.

### SPEC-014 [PASS]
- **Checklist item**: Edge cases covered
- **Requirement**: FR-044, FR-047, FR-049
- **File**: .github/skills/plan-api-contracts/SKILL.md#L50-L55, .github/skills/plan-state-machines/SKILL.md#L53-L59, .github/skills/plan-error-catalogs/SKILL.md#L50-L55
- **Description**: All three skills handle the edge case where the spec companion artifact does not exist by deriving from spec prose and noting the derivation source in a comment. No regression from round 1.

### SPEC-015 [PASS]
- **Checklist item**: Edge cases covered
- **Requirement**: FR-045
- **File**: .github/skills/plan-api-contracts/SKILL.md#L163-L175
- **Description**: Error code applicability rules correctly scope error codes per endpoint type: 400 for endpoints with request body/params, 401 for authenticated endpoints, 403 for role-based endpoints, 404 for resource-by-ID endpoints, 409 for creation with uniqueness, 422 for complex validation, 500 for all endpoints. No regression from round 1.

### SPEC-016 [PASS]
- **Checklist item**: Edge cases covered
- **Requirement**: FR-046
- **File**: .github/skills/plan-state-machines/SKILL.md#L49-L52, .github/skills/plan-state-machines/SKILL.md#L165
- **Description**: Correctly handles WPs without stateful entities: Step 1 skip rules exclude WPs that don't create/modify entities with state fields, and Constraints section explicitly states "Do NOT generate files for WPs without stateful entities." No regression from round 1.

### SPEC-017 [N/A]
- **Checklist item**: Data model match
- **Justification**: These skills are declarative instruction files (SKILL.md) that generate contract artifacts. They do not define or implement data model fields themselves. Data model compliance is the responsibility of the plan-data-schemas skill (WP17).

### SPEC-018 [N/A]
- **Checklist item**: API contract match
- **Justification**: These skills do not expose API endpoints themselves. They provide instructions for generating API contract definitions. The generated artifacts would be validated at runtime, not in this review.

### SPEC-019 [N/A]
- **Checklist item**: Error codes match
- **Justification**: These skills do not return runtime error codes. They provide instructions for generating error catalog definitions. Error code accuracy depends on the spec artifacts consumed at generation time.

### SPEC-020 [N/A]
- **Checklist item**: Success criteria verification
- **Justification**: WP18 does not reference specific SC-XXX items. The WP's "Independent Test" describes a runtime integration test (dispatch skills against a plan accumulator) that cannot be verified through static review. Deferred verification: requires runtime execution of the planner coordinator.

### SPEC-021 [PASS]
- **Checklist item**: FR classification - Error paths handled
- **Requirement**: FR-043, FR-046, FR-048
- **File**: .github/skills/plan-api-contracts/SKILL.md#L43-L55, .github/skills/plan-state-machines/SKILL.md#L43-L59, .github/skills/plan-error-catalogs/SKILL.md#L43-L62
- **Description**: All three skills handle the primary error/fallback path: missing spec companion artifacts. plan-api-contracts and plan-state-machines include skip rules for non-applicable WPs. plan-error-catalogs includes skip rules for WPs without error paths. These are the only error conditions relevant to declarative skill instructions. No regression from round 1.
