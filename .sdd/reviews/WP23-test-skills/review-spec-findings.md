---
skill: review-spec
wp: WP23-test-skills
spec: .sdd/specs/004-coder-v2.spec.md
reviewed_at: 2026-04-05T15:00:00Z
status: completed
finding_counts:
  pass: 17
  warn: 0
  fail: 0
  na: 3
files_reviewed:
  - .github/skills/code-unit-tests/SKILL.md
  - .github/skills/code-integration-tests/SKILL.md
  - .github/skills/CODER-SKILL-CONTRACT.md
  - .sdd/plans/WP23-test-skills.md
  - .sdd/specs/004-coder-v2.spec.md
---

# review-spec Findings for WP23-test-skills

## Summary

Evaluated 13 in-scope FRs (FR-017, FR-018, FR-019, FR-027 through FR-033) plus success criteria SC-004 and SC-005 across two implementation files: `code-unit-tests/SKILL.md` and `code-integration-tests/SKILL.md`. All FRs are classified as **Compliant**. Both skills faithfully translate the spec's FR obligations into actionable AI subagent instructions with correct input/output contracts, proper BDD test derivation logic, coverage enforcement, and integration test boundary handling. No deviations or missing requirements found.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-017 (Skill Input Contract)
- **File**: .github/skills/code-unit-tests/SKILL.md#L18-L28
- **Description**: Input contract table lists all 8 required inputs (skill_path, wp_path, contracts_dir, spec_path, patterns, target_language, target_framework, task_list) matching FR-017 and CODER-SKILL-CONTRACT.md exactly.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-017 (Skill Input Contract)
- **File**: .github/skills/code-integration-tests/SKILL.md#L18-L28
- **Description**: Input contract table lists all 8 required inputs matching FR-017 and CODER-SKILL-CONTRACT.md exactly.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-018 (Execution Sequence)
- **File**: .github/skills/code-unit-tests/SKILL.md#L32-L38
- **Description**: Execution sequence follows the 5-step order from FR-018: (1) Read SKILL.md, (2) Read WP + contracts, (3) Read spec sections, (4) Execute test writing, (5) Report results.

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-018 (Execution Sequence)
- **File**: .github/skills/code-integration-tests/SKILL.md#L32-L38
- **Description**: Execution sequence follows the 5-step order from FR-018 identically.

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation + postconditions
- **Requirement**: FR-019 (Output Contract)
- **File**: .github/skills/code-unit-tests/SKILL.md#L42-L60
- **Description**: Output contract includes all 6 required fields (status, files_modified, tasks_completed, test_results, issues, failure_reason) with correct types and constraints. test_results includes pass_count, fail_count, coverage_pct. Explicitly documents that test_results.fail_count drives debug dispatch (FR-010).

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation + postconditions
- **Requirement**: FR-019 (Output Contract)
- **File**: .github/skills/code-integration-tests/SKILL.md#L42-L60
- **Description**: Output contract matches FR-019 exactly with all 6 fields. test_results includes pass_count, fail_count, coverage_pct. Explicitly documents fail_count drives debug dispatch.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation (all 5 sub-requirements)
- **Requirement**: FR-027 (Unit Test Writing)
- **File**: .github/skills/code-unit-tests/SKILL.md#L64-L195
- **Description**: All 5 FR-027 sub-requirements are fully addressed: (1) BDD derivation from spec scenarios in Step 1 with explicit "do NOT derive from implementation" instruction; (2) Happy/error/edge coverage in Step 2a with table of scenario types; (3) Real behavior testing in Step 2b with code examples; (4) External-only mocking in Step 2c with clear mock boundary rule; (5) Project test framework detection in Step 2d with language-specific table.

### SPEC-008 [PASS]
- **Checklist item**: FR classification - SHALL NOT obligations
- **Requirement**: FR-028 (Test Validity)
- **File**: .github/skills/code-unit-tests/SKILL.md#L197-L270
- **Description**: All 3 FR-028 sub-requirements are fully addressed: (1) Trivial assertions forbidden in Step 3a with multi-language examples; (2) Empty test bodies forbidden in Step 3b with multi-language examples; (3) Mock-only assertions forbidden in Step 3c with valid alternative shown. Self-check section (3d) adds value beyond the spec.

### SPEC-009 [PASS]
- **Checklist item**: FR classification - SHALL obligation + postconditions
- **Requirement**: FR-029 (Test Execution and Reporting)
- **File**: .github/skills/code-unit-tests/SKILL.md#L300-L340
- **Description**: Step 5 covers running all tests with coverage enabled (language-specific commands in 5a), capturing results (5b), and reporting to coordinator in FR-019 format (5c). Pass/fail counts and coverage percentage are all captured.

### SPEC-010 [PASS]
- **Checklist item**: FR classification - SHALL obligation + error paths
- **Requirement**: FR-030 (Coverage Thresholds)
- **File**: .github/skills/code-unit-tests/SKILL.md#L344-L400
- **Description**: Step 6 enforces exactly 80% code coverage and 90% branch coverage (6a). Below-threshold behavior adds targeted tests for uncovered lines/branches with priority on branch coverage (6b). Includes pragmatic 3-iteration safety limit (6c) with proper escalation via issues field. Thresholds match spec exactly.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation (all 4 sub-requirements)
- **Requirement**: FR-031 (Integration Test Boundaries)
- **File**: .github/skills/code-integration-tests/SKILL.md#L64-L200
- **Description**: All 4 FR-031 sub-requirements are fully addressed: (1) Cross-module boundary tests in Step 2a with code example; (2) Real database/storage tests in Step 2b with lifecycle testing; (3) External API mock tests using contract definitions in Step 2c with contract compliance rule; (4) Data setup/teardown in Step 2d with framework-specific patterns and isolation rule.

### SPEC-012 [PASS]
- **Checklist item**: FR classification - SHALL obligation (all 3 sub-requirements)
- **Requirement**: FR-032 (External Dependencies)
- **File**: .github/skills/code-integration-tests/SKILL.md#L178-L245
- **Description**: All 3 FR-032 sub-requirements are fully addressed: (1) Contract-based mock responses in Step 2c with explicit "SHALL NOT use arbitrary test data" constraint; (2) Timeout/retry/error handling in Step 2e with comprehensive failure mode table; (3) Contract schema verification in Step 2f with request/response validation.

### SPEC-013 [PASS]
- **Checklist item**: FR classification - SHALL obligation + postconditions
- **Requirement**: FR-033 (Integration Test Execution)
- **File**: .github/skills/code-integration-tests/SKILL.md#L255-L295
- **Description**: Step 4 covers running integration tests separately (4a with language-specific commands), capturing pass/fail counts (4b), and reporting to coordinator in FR-019 format (4c). Failed tests reported via fail_count for debug dispatch.

### SPEC-014 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-004 (Tests pass with coverage thresholds)
- **File**: .github/skills/code-unit-tests/SKILL.md#L344-L400
- **Description**: The unit test skill enforces the exact thresholds from SC-004: 80% code coverage and 90% branch coverage. The skill self-iterates to meet thresholds before reporting success.

### SPEC-015 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-005 (Single skill file addition)
- **File**: .github/skills/code-unit-tests/SKILL.md#L1-L5
- **Description**: Both skills are self-contained in single SKILL.md files within `.github/skills/code-<name>/` directories, matching the pattern required by SC-005. No coordinator modification needed.

### SPEC-016 [PASS]
- **Checklist item**: FR classification - compliance with common contract
- **Requirement**: CODER-SKILL-CONTRACT.md (Sections 6, 8)
- **File**: .github/skills/code-unit-tests/SKILL.md#L1-L5
- **Description**: Both skills have valid YAML frontmatter with `name`, `description`, and `argument-hint` fields. Both reference the common contract. Both include Constraints sections covering SHALL NOT rules (no implementation-derived tests, no self-review, no contract modification, no test deletion).

### SPEC-017 [PASS]
- **Checklist item**: FR classification - coordinator integration
- **Requirement**: FR-005, FR-010 (Skill Discovery and Debug Dispatch)
- **File**: .github/skills/code-unit-tests/SKILL.md#L55-L60
- **Description**: Both skills are discoverable via glob `.github/skills/code-*/SKILL.md`. Output contract's `test_results.fail_count` field is explicitly documented as the trigger for debug skill dispatch (FR-010), ensuring correct coordinator integration.

### SPEC-018 [N/A]
- **Checklist item**: Data model match
- **Justification**: No data models in this WP. Both artifacts are markdown instruction files for AI subagents, not executable code with data entities.

### SPEC-019 [N/A]
- **Checklist item**: API contract match
- **Justification**: No API endpoints in this WP. Both artifacts are markdown instruction files that describe how to write and run tests, not API implementations.

### SPEC-020 [N/A]
- **Checklist item**: Error codes match
- **Justification**: No error codes defined in this WP's scope. Error handling is described as instruction prose (e.g., "report failure to coordinator") rather than as coded error responses.
