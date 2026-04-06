---
lane: doing
---

# WP44 - Configurable Coverage Thresholds

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P2 |
| Lane | planned |
| Depends on | WP41 |
| Goal | Allow per-WP coverage threshold overrides via frontmatter fields, with defaults of 80% code and 90% branch |
| Status | Not Started |
| Independent Test | Create a WP with `coverage_code: 60` and `coverage_branch: 70` in frontmatter. Read the code-unit-tests SKILL.md and verify it documents reading these fields with fallback to 80/90 defaults. |
| Parallelisable | Yes (with WP45) |
| Prompt | `.sdd/plans/WP44-coverage-thresholds.md` |

## Objective

Add `coverage_code` and `coverage_branch` optional integer frontmatter fields to WP files and update the code-unit-tests, code-env-setup, and spec-test-strategy skills to read them with fallback to defaults (80%/90%). This lets pipeline operators set project-appropriate coverage thresholds per-WP without modifying skill instructions.

## Spec References

FR-034, FR-035, FR-036, FR-037, FR-038, FR-039, Section 7.1 (WP Frontmatter Extended), Section 8.1 (coverage_code/coverage_branch read), Section 6.4 (Coverage Threshold Override flow), US-04

## Tasks

### T44-01 - Define coverage_code frontmatter field

- **Description**: Document `coverage_code` as an optional integer field (range 0-100) in the WP frontmatter schema with default value 80.
- **Spec refs**: FR-034, Section 7.1
- **Parallel**: Yes (with T44-02)
- **Acceptance criteria**:
  - [x] WP frontmatter schema accepts `coverage_code` as a valid optional field of type integer (FR-034)
  - [x] Valid range is 0-100 inclusive
  - [x] If `coverage_code` is present but out of range or not an integer, the reading skill SHALL halt with: "Invalid coverage_code value '<value>'. Must be an integer 0-100." (FR-034)
  - [x] Default value when absent is 80
- **Test requirements**: content, BDD (US-04 Scenario 3)
- **Depends on**: none
- **Implementation Guidance**:
  - Error E-004 (COVERAGE_OUT_OF_RANGE): halt with descriptive error
  - A value of 0 IS valid (no coverage required for prototyping WPs)
  - This is a schema definition -- update WP template documentation

### T44-02 - Define coverage_branch frontmatter field

- **Description**: Document `coverage_branch` as an optional integer field (range 0-100) in the WP frontmatter schema with default value 90.
- **Spec refs**: FR-035, Section 7.1
- **Parallel**: Yes (with T44-01)
- **Acceptance criteria**:
  - [x] WP frontmatter schema accepts `coverage_branch` as a valid optional field of type integer (FR-035)
  - [x] Valid range is 0-100 inclusive
  - [x] Same validation and halt behavior as `coverage_code` (FR-035)
  - [x] Default value when absent is 90
- **Test requirements**: content
- **Depends on**: none
- **Implementation Guidance**:
  - Same error behavior as coverage_code (E-004)
  - Each field is independent -- specifying one does not require specifying the other

### T44-03 - Update code-unit-tests skill for configurable thresholds

- **Description**: Update `.github/skills/code-unit-tests/SKILL.md` to read `coverage_code` and `coverage_branch` from WP frontmatter and use them as enforcement thresholds, falling back to 80/90 defaults.
- **Spec refs**: FR-036, FR-037, Section 6.4 (steps 6-8)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] code-unit-tests skill reads `coverage_code` and `coverage_branch` from WP frontmatter (FR-036)
  - [x] Coverage thresholds used for enforcement match WP frontmatter values (FR-036)
  - [x] When frontmatter fields are absent, skill uses defaults: 80% code, 90% branch (FR-037)
  - [x] Given a WP with `coverage_code: 60` and `coverage_branch: 70`, the skill enforces 60/70 (US-04 Scenario 1)
  - [x] Given a WP with no coverage fields, the skill enforces 80/90 (US-04 Scenario 2)
  - [x] Given `coverage_code: 60` but no `coverage_branch`, the skill uses 60/90 (US-04 Edge Case 1)
- **Test requirements**: BDD (US-04 Scenarios 1-3, Edge Case 1), content
- **Depends on**: T44-01, T44-02
- **Implementation Guidance**:
  - Find the coverage threshold section in code-unit-tests/SKILL.md
  - Replace hardcoded 80/90 values with: "Read `coverage_code` from WP frontmatter (default 80 if absent)"
  - Add validation: halt if value is out of range [0, 100]
  - Files to modify: `.github/skills/code-unit-tests/SKILL.md`

### T44-04 - Update code-env-setup skill for configurable thresholds

- **Description**: Update `.github/skills/code-env-setup/SKILL.md` to read `coverage_code` and `coverage_branch` from WP frontmatter when configuring coverage tooling, with the same fallback defaults.
- **Spec refs**: FR-038, FR-037, Section 6.4 (steps 3-5)
- **Parallel**: Yes (with T44-03)
- **Acceptance criteria**:
  - [x] code-env-setup skill reads `coverage_code` and `coverage_branch` from WP frontmatter (FR-038)
  - [x] Coverage tool configuration uses WP-specific or default thresholds (FR-038)
  - [x] Fallback defaults match: 80% code, 90% branch (FR-037)
- **Test requirements**: content
- **Depends on**: T44-01, T44-02
- **Implementation Guidance**:
  - Find the coverage tooling setup section in code-env-setup/SKILL.md
  - Add frontmatter reading logic with same defaults and validation as T44-03
  - Files to modify: `.github/skills/code-env-setup/SKILL.md`

### T44-05 - Update spec-test-strategy skill for configurable references

- **Description**: Update `.github/skills/spec-test-strategy/SKILL.md` to state that coverage thresholds are configurable per-WP via frontmatter fields, with defaults of 80% code and 90% branch.
- **Spec refs**: FR-039
- **Parallel**: Yes (with T44-03, T44-04)
- **Acceptance criteria**:
  - [x] spec-test-strategy skill states coverage thresholds are configurable per-WP (FR-039)
  - [x] References `coverage_code` and `coverage_branch` frontmatter fields
  - [x] States defaults of 80% code and 90% branch
  - [x] No hardcoded values appear without mention of configurability
- **Test requirements**: content
- **Depends on**: none
- **Implementation Guidance**:
  - Find the coverage thresholds section in spec-test-strategy/SKILL.md
  - Change from "80% code coverage and 90% branch coverage" to "configurable per-WP via `coverage_code` (default 80) and `coverage_branch` (default 90) frontmatter fields"
  - Files to modify: `.github/skills/spec-test-strategy/SKILL.md`

### T44-06 - Verify threshold consistency across skills

- **Description**: Verify that code-unit-tests, code-env-setup, and spec-test-strategy all reference the same default values and frontmatter field names.
- **Spec refs**: FR-034-039, Section 11.3
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [x] All three skills reference `coverage_code` and `coverage_branch` by the same field names
  - [x] All three use the same defaults: 80 and 90
  - [x] Validation behavior is consistent (halt on out-of-range)
- **Test requirements**: integration (cross-file consistency)
- **Depends on**: T44-03, T44-04, T44-05
- **Implementation Guidance**:
  - Cross-reference Section 11.3: "WP frontmatter fields -> Agent instructions"

## Implementation Notes

- All deliverables are markdown skill instruction files -- no executable code
- Coverage threshold overrides are per-WP (Design Decision 4, Section 9.4) -- not per-spec or per-project
- Each field is independent: specifying only `coverage_code` uses the override for code and the default for branch
- A value of 0 is valid (no coverage enforcement) to support prototyping WPs
- This WP depends on WP41 because coverage fields are part of the same WP frontmatter extension pattern

## Risks & Mitigations

- **Risk**: Skills may have deeply embedded hardcoded threshold values. **Mitigation**: Search each skill file for "80" and "90" to find all threshold references.
- **Risk**: Other skills beyond those listed may also reference coverage thresholds. **Mitigation**: Grep all SKILL.md files for coverage-related terms.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-07T00:00:00Z - coder - lane=doing - Starting implementation
