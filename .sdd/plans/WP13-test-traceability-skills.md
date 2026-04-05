---
lane: done
---

# WP13 - Test Strategy & Traceability Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/002-spec-architect-v2.spec.md` |
| Priority | P2 |
| Lane | planned |
| Depends on | WP08, WP09 |
| Goal | Implement the spec-test-strategy and spec-traceability skills that produce BDD scenarios, test requirements, traceability matrix, glossary, and version history |
| Status | Complete |
| Independent Test | Dispatch spec-test-strategy and spec-traceability against a full accumulator (sections 1-10.2). Verify: Section 11 has BDD scenarios for every acceptance criterion from Section 5; sections 14-18 written; traceability matrix has no empty cells; no orphan FRs or USes |
| Parallelisable | Yes (with WP10, WP11, WP12 after WP09 completes) |
| Prompt | `.sdd/plans/WP13-test-traceability-skills.md` |

## Objective

Implement the final two skills in the canonical order. spec-test-strategy produces Section 11 (Test Requirements) with BDD scenarios mapped 1:1 to acceptance criteria, unit/integration/E2E/performance/security test requirements, and coverage thresholds. spec-traceability produces sections 14-18 (Open Questions, Glossary, Traceability Matrix, Technical References, Version History) and performs final cross-spec validation to catch orphan FRs/USes and traceability gaps.

## Spec References

- FR-023 through FR-028 (common skill contract)
- FR-050 through FR-052 (spec-test-strategy skill)
- FR-053 through FR-055 (spec-traceability skill)
- Section 4.9 (Test Strategy Skill specification)
- Section 4.10 (Traceability Skill specification)

## Tasks

### T13-01 - Implement spec-test-strategy SKILL.md

- **Description**: Replace the stub SKILL.md in `.github/skills/spec-test-strategy/` with the full skill implementation. The skill produces Section 11 (Test Requirements).
- **Spec refs**: FR-050
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md contains instructions for Section 11 with subsections:
    - 11.1 Unit Tests: modules requiring coverage, minimum thresholds (80% code, 90% branch), specific edge cases
    - 11.2 BDD / Acceptance Tests: Gherkin scenarios for every acceptance criterion from Section 5
    - 11.3 Integration Tests: component boundaries, external dependency mocking, data setup/teardown
    - 11.4 End-to-End Tests: critical user journeys, target environments, tools
    - 11.5 Performance Tests: scenarios, thresholds
    - 11.6 Security Tests: OWASP checks, auth/authz test cases
  - [x] Skill reads accumulator (sections 1-10.2) to derive test scenarios from requirements, stories, data model, API, architecture, and security
  - [x] Coverage thresholds specified: 80% code coverage, 90% branch coverage minimum
- **Test requirements**: BDD (Scenario 1 from Section 11.2)
- **Depends on**: T08-02 (stub exists)
- **Implementation Guidance**:
  - Gherkin scenario format:
    ```gherkin
    Feature: <Feature area from Section 4>

      Scenario: <Acceptance scenario title from Section 5>
        Given <precondition from US-XX>
        When <action>
        Then <expected result>
        And <additional assertion>
    ```
  - Each acceptance scenario from Section 5 maps to exactly one Gherkin scenario in Section 11.2
  - Unit test guidance: identify pure functions, data transformations, validation logic
  - Integration test guidance: identify component boundaries (e.g., service -> database, service -> external API)
  - Coverage tool recommendations by language: pytest-cov (Python), istanbul/c8 (Node/TypeScript)

### T13-02 - Add 1:1 BDD scenario mapping requirement

- **Description**: Add explicit instructions requiring a 1:1 mapping between acceptance scenarios (Section 5 user stories) and Gherkin scenarios (Section 11.2).
- **Spec refs**: FR-051
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Skill instructions mandate: every acceptance scenario in every US from Section 5 has exactly one Gherkin scenario in Section 11.2
  - [x] After writing Section 11.2, the skill reports any missing mappings
  - [x] Edge cases from Section 5 also have corresponding test scenarios
  - [x] Format: each Gherkin scenario references its source US (e.g., "# Source: US-01 Scenario 2")
- **Test requirements**: BDD (verification step within skill)
- **Depends on**: T13-01
- **Implementation Guidance**:
  - Add validation instructions:
    ```markdown
    ## BDD Mapping Validation (MANDATORY)
    After writing Section 11.2:
    1. Count acceptance scenarios in Section 5 (each Given/When/Then block)
    2. Count Gherkin scenarios in Section 11.2
    3. The counts MUST match. If not, add missing scenarios.
    4. Each Gherkin scenario header MUST reference its source: "# Source: US-XX Scenario N"
    ```

### T13-03 - Add BDD/TDD emphasis

- **Description**: Add instructions emphasizing that tests derive from spec acceptance scenarios, not from implementation details.
- **Spec refs**: FR-052
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Skill instructions state: "Tests derive from acceptance scenarios (Section 5), NOT from implementation"
  - [x] Test descriptions reference spec behavior, not internal function names
  - [x] Coverage thresholds are stated explicitly (80% code, 90% branch)
  - [x] Test design principle: "Write the test an acceptance scenario describes, not the test an implementation suggests"
- **Test requirements**: none (process requirement)
- **Depends on**: T13-01
- **Implementation Guidance**:
  - Add emphasis section:
    ```markdown
    ## BDD/TDD Principle
    ALL tests in this section derive from spec acceptance scenarios (Section 5) and
    functional requirements (Section 4), NOT from implementation details.

    - Test names describe BEHAVIOR, not functions: "User can register with valid email" not "test_create_user_service"
    - Test assertions verify SPEC POSTCONDITIONS, not internal state
    - Coverage thresholds: 80% code coverage, 90% branch coverage (minimum)
    ```

### T13-04 - Add common skill contract compliance to test-strategy skill

- **Description**: Ensure spec-test-strategy fully complies with the common skill contract.
- **Spec refs**: FR-023, FR-024, FR-025, FR-026, FR-027
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Input contract, execution sequence, output format, modification constraints
  - [x] This skill produces no artifacts; step 5 is "N/A"
- **Test requirements**: none (contract compliance)
- **Depends on**: T13-01
- **Implementation Guidance**:
  - Same pattern as T10-03. This skill reads the most accumulator content (sections 1-10.2).

### T13-05 - Implement spec-traceability SKILL.md

- **Description**: Replace the stub SKILL.md in `.github/skills/spec-traceability/` with the full skill implementation. The skill produces sections 14-18.
- **Spec refs**: FR-053
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md contains instructions for:
    - Section 14 (Open Questions): unresolved decisions with impact and owner
    - Section 15 (Glossary): key terms, acronyms, domain concepts defined
    - Section 16 (Traceability Matrix): table mapping FR -> US -> Acceptance Scenario -> Test Type -> Test Section Ref
    - Section 17 (Technical References): sources grouped by topic with URLs and dates
    - Section 18 (Version History): initial version entry
  - [x] Skill reads the ENTIRE accumulator (all sections 1-11) to build the matrix
  - [x] Skill builds the matrix by scanning: Section 4 for FRs, Section 5 for USes, Section 11 for tests
- **Test requirements**: BDD (Scenario 1 from Section 11.2)
- **Depends on**: T08-02 (stub exists)
- **Implementation Guidance**:
  - Traceability matrix format:
    ```markdown
    | FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
    |-------|-------------------|------------|--------------------|-----------|----|
    | FR-001 | Brief selection | US-01 | Scenario 1 | BDD | 11.2 |
    ```
  - Glossary format: alphabetically sorted definition list
  - Technical references: group by topic (Architecture, Security, Standards, etc.)
  - Version history: `| 1.0 | <date> | Spec Architect | Initial specification |`

### T13-06 - Add traceability matrix validation with no empty cells

- **Description**: Add instructions for the traceability matrix to have no empty cells, with filling logic and gap reporting.
- **Spec refs**: FR-054
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Skill instructions mandate: NO empty cells in the traceability matrix
  - [x] For every FR: at least one US mapping, one acceptance scenario, one test type, one test section ref
  - [x] If any cell is empty, skill attempts to fill by cross-referencing sections
  - [x] If unable to fill, skill adds a `[TRACEABILITY GAP]` marker (not an empty cell)
- **Test requirements**: BDD (Scenario 2 from Section 11.2 -- no empty cells in matrix)
- **Depends on**: T13-05
- **Implementation Guidance**:
  - Validation logic:
    ```markdown
    ## Matrix Completeness Validation (MANDATORY)
    After building the traceability matrix:
    1. Scan every row. If any cell is empty:
       a. Cross-reference Section 4 (FRs), Section 5 (USes), Section 11 (tests)
       b. If a mapping exists, fill the cell
       c. If no mapping exists, write [TRACEABILITY GAP: FR-XXX has no <missing column>]
    2. Report total number of gaps found
    3. Zero gaps = complete matrix
    ```

### T13-07 - Add cross-spec validation for orphan FRs and USes

- **Description**: Add validation that checks for orphan FRs (not referenced by any US), orphan USes (not referencing any FR), and orphan Gherkin scenarios (not mapped to any acceptance scenario).
- **Spec refs**: FR-055
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Skill validates: every FR-XXX in Section 4 is referenced by at least one entry in the traceability matrix
  - [x] Skill validates: every US-XX in Section 5 is referenced by at least one entry in the matrix
  - [x] Skill validates: every Gherkin scenario in Section 11.2 maps to an acceptance scenario in Section 5
  - [x] Orphan items reported with `[TRACEABILITY GAP: <description>]` markers
  - [x] No duplicate assignments (one FR should not appear in two unrelated US mappings without justification)
- **Test requirements**: BDD (verification within skill output)
- **Depends on**: T13-05
- **Implementation Guidance**:
  - Validation steps:
    ```markdown
    ## Orphan Detection (MANDATORY)
    After completing the traceability matrix:
    1. List all FR-XXX from Section 4. Check each appears in at least one matrix row.
       Flag missing: [TRACEABILITY GAP: FR-XXX has no US mapping]
    2. List all US-XX from Section 5. Check each is referenced in at least one matrix row.
       Flag missing: [TRACEABILITY GAP: US-XX not in matrix]
    3. List all Gherkin Feature/Scenario in Section 11.2. Check each maps to Section 5.
       Flag missing: [TRACEABILITY GAP: Gherkin scenario "X" has no acceptance scenario source]
    ```

### T13-08 - Add common skill contract compliance to traceability skill

- **Description**: Ensure spec-traceability fully complies with the common skill contract.
- **Spec refs**: FR-023, FR-024, FR-025, FR-026, FR-027
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Input contract, execution sequence, output format, modification constraints
  - [x] This skill produces no artifacts; step 5 is "N/A"
  - [x] This skill reads the entire accumulator (all sections 1-11) -- documented as most
- **Test requirements**: none (contract compliance)
- **Depends on**: T13-05
- **Implementation Guidance**:
  - Same contract compliance pattern. This is the last skill in the chain and reads the MOST context.

### T13-09 - Test both skills with full accumulator

- **Description**: Manually test both skills against a full accumulator (sections 1-10.2) to verify completeness.
- **Spec refs**: All FR-050 through FR-055
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Dispatch spec-test-strategy: verify Section 11 with all 6 subsections
  - [x] Verify 1:1 Gherkin-to-acceptance-scenario mapping
  - [x] Verify coverage thresholds stated (80% code, 90% branch)
  - [x] Dispatch spec-traceability: verify sections 14-18
  - [x] Verify traceability matrix has no empty cells
  - [x] Verify no orphan FRs or USes
  - [x] Verify no prior sections modified
- **Test requirements**: integration (manual invocation)
- **Depends on**: T13-01 through T13-08
- **Implementation Guidance**:
  - Use output from WP10-WP12 testing as the full test accumulator
  - This is the final validation: the accumulator should now contain all 16+ sections
  - After this test, the coordinator's post-completion validation (WP09) should also pass

## Implementation Notes

- spec-test-strategy reads sections 1-10.2 (including security). It needs requirements, stories, data model, API, and security context to write comprehensive test requirements.
- spec-traceability reads ALL sections (1-11). It is the last skill and performs final cross-validation.
- Neither skill produces companion artifacts. Both produce prose only.
- The traceability matrix is the key quality gate: a complete matrix with no gaps proves the spec is internally consistent.
- After these skills complete, the coordinator runs post-completion validation (FR-017, FR-018) as a final check.

## Parallel Opportunities

- T13-01 (test strategy) and T13-05 (traceability) are separate skill files and can be developed in parallel.
- T13-03, T13-04 can be developed in parallel with T13-02.
- T13-07, T13-08 can be developed in parallel with T13-06.
- T13-09 (testing) must be last.

## Risks & Mitigations

- **Risk**: 1:1 BDD mapping is tedious and the LLM skips scenarios.
  - **Mitigation**: Mandatory validation step counts scenarios and reports mismatches.
- **Risk**: Traceability matrix has too many rows for large specs, overwhelming the context window.
  - **Mitigation**: Skill processes FRs in batches if needed; matrix is a simple table that scales linearly.
- **Risk**: Orphan detection misses FRs hidden in nested subsections.
  - **Mitigation**: Skill scans for the regex pattern `FR-\d{3}` across the entire Section 4, not just top-level items.

## Self-Review

**Spec Compliance**: Both skills implement all FRs (FR-050 through FR-055). spec-test-strategy covers all 6 subsections (11.1-11.6), 1:1 BDD mapping (FR-051), BDD/TDD emphasis (FR-052), and coverage thresholds. spec-traceability covers all 5 sections (14-18), no-empty-cells rule (FR-054), and orphan detection (FR-055).

**Correctness**: spec-test-strategy produces Section 11 with unit tests, BDD scenarios (1:1 mapped to acceptance criteria), integration tests, E2E tests, performance tests, and security tests. spec-traceability produces sections 14-18 with open questions, glossary, traceability matrix (no empty cells), technical references, and version history with mandatory orphan detection.

**Code quality**: Both files encoding-clean. spec-test-strategy: 211 lines, spec-traceability: 220 lines.

**Scope discipline**: Only two files modified. No unrelated changes.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T16:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T16:30:00Z - coder - lane=for_review - All 9 tasks complete, both skills implemented, submitted for review
- 2026-04-05T17:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARN)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T17:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (PASS), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (PASS), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked off for 9 tasks (T13-01 through T13-09). All FR-050 through FR-055 obligations satisfied.
- [PASS] Activity Log: Consistent lane transitions: planned -> doing -> for_review.
- [WARN] Commit granularity: Single implementation commit (21b8662) covers both skill files and all 9 tasks. Expected granular commits per task or logical grouping.
- [PASS] Encoding: No prohibited Unicode characters found in either implementation file.

### Review Feedback

> No FAIL findings. No action items required.

(No FB-XX items -- zero FAILs across all dimensions.)

### Warnings
- [WARN] Commit granularity: Single bulk commit `21b8662 feat(skills): implement spec-test-strategy and spec-traceability skills (WP13)` covers all 9 tasks across 2 files. Prefer separate commits per skill file or per logical task group. (Process compliance PROC-003)

### Cross-Correlation Notes
No cross-correlation findings. No duplicates, conflicts, or systemic patterns detected across skill findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 13 | 0 | 0 |
| review-security | 0 | 0 | 0 |
| review-quality | 4 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 2 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 0 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **22** | **1** | **0** |
