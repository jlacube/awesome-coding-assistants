---
lane: done
---

# WP23 - Test Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/004-coder-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP20, WP21 |
| Goal | Implement the code-unit-tests and code-integration-tests SKILL.md files that write and run tests, enforce coverage thresholds, and report results to the coordinator |
| Status | Not Started |
| Independent Test | Invoke the Coder on a WP after implementation. Verify: code-unit-tests writes BDD-derived unit tests covering happy + error + edge paths; code-integration-tests writes boundary and mock tests; both run tests and report results with coverage; coverage meets 80% code / 90% branch |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP23-test-skills.md` |

## Objective

Implement the two testing skills in the Coder V2 pipeline: `code-unit-tests` (Phase 3) and `code-integration-tests` (Phase 4). Unit tests are derived from spec acceptance scenarios using a BDD approach and cover happy paths, error paths, and edge cases. Integration tests verify component boundaries, mock external dependencies using contract schemas, and include data lifecycle management. Both skills run their tests, verify results, and report to the coordinator. These skills provide the verification phase that triggers debug dispatch on failure.

## Spec References

FR-027, FR-028, FR-029, FR-030 (code-unit-tests), FR-031, FR-032, FR-033 (code-integration-tests), FR-017, FR-018, FR-019 (common skill contract), Section 4.5 (Unit Tests), Section 4.6 (Integration Tests), Section 8.2 (Skill Prompt Template)

## Tasks

### T23-01 - Create code-unit-tests SKILL.md structure

- **Description**: Replace the stub code-unit-tests SKILL.md with the full skill file. Write the YAML frontmatter, input contract table (per CODER-SKILL-CONTRACT.md), execution sequence, and output format sections.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md has valid YAML frontmatter with name `code-unit-tests` and description matching FR-006
  - [x] Input contract table lists all 8 inputs from FR-017
  - [x] Execution sequence follows FR-018
  - [x] Output format matches FR-019: status, files_modified, tasks_completed, test_results (pass_count, fail_count, coverage_pct), issues, failure_reason
  - [x] Common contract reference to `.github/skills/CODER-SKILL-CONTRACT.md` is included
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Same structure as code-env-setup SKILL.md (T22-01)
  - Files to modify: `.github/skills/code-unit-tests/SKILL.md`

### T23-02 - Write unit test generation logic

- **Description**: Write the code-unit-tests skill instructions for generating unit tests. Tests SHALL be derived from spec acceptance scenarios (BDD approach), cover happy path + error paths + edge cases per task, and mock only external dependencies.
- **Spec refs**: FR-027
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Tests SHALL derive from spec acceptance scenarios (BDD approach), not from implementation details (FR-027.1)
  - [x] Tests SHALL cover happy path + error paths + edge cases per task (FR-027.2)
  - [x] Tests SHALL test real behavior through real code paths -- no vacuous assertions (FR-027.3)
  - [x] Tests SHALL mock only external dependencies, never the subject under test (FR-027.4)
  - [x] Tests SHALL use the project's test framework (pytest, Jest, etc.) (FR-027.5)
- **Test requirements**: BDD
- **Depends on**: T23-01
- **Implementation Guidance**:
  - Pattern: For each task in the WP, read its acceptance scenarios and create tests that verify each scenario's Given/When/Then
  - Known pitfalls: Tests must be derived from spec scenarios, NOT from reading the implementation code. This prevents tests that merely confirm what the code does.
  - Official docs: https://cucumber.io/docs/bdd/

### T23-03 - Write test validity rules

- **Description**: Write the code-unit-tests skill constraints that prevent trivial or vacuous test assertions. Every test must be capable of failing.
- **Spec refs**: FR-028
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] The skill SHALL NOT produce `assert True` or equivalent trivial assertions (FR-028.1)
  - [x] The skill SHALL NOT produce empty test bodies or `pass` stubs (FR-028.2)
  - [x] The skill SHALL NOT produce tests that merely confirm a mock's return value (FR-028.3)
  - [x] Every test SHALL be capable of failing
- **Test requirements**: none
- **Depends on**: T23-01
- **Implementation Guidance**:
  - Pattern: Write explicit "SHALL NOT" rules in the SKILL.md with examples of bad patterns
  - Known pitfalls: `assert mock.called` is borderline -- acceptable only when verifying a side effect, not when testing return values

### T23-04 - Write test execution and coverage threshold enforcement

- **Description**: Write the code-unit-tests skill instructions for running all tests after writing them, reporting results, and verifying coverage meets thresholds. If coverage is below thresholds, the skill SHALL add more tests.
- **Spec refs**: FR-029, FR-030
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL run all unit tests after writing them and report results (pass/fail counts, coverage percentage) (FR-029)
  - [x] The skill SHALL verify code coverage is at least 80% (FR-030.1)
  - [x] The skill SHALL verify branch coverage is at least 90% (FR-030.2)
  - [x] If coverage is below thresholds, the skill SHALL add more tests targeting uncovered lines/branches (FR-030)
  - [x] Given unit tests pass but coverage is 72% (below 80%), the skill adds more tests and re-checks until coverage meets thresholds (BDD Scenario 8)
- **Test requirements**: BDD
- **Depends on**: T23-02
- **Implementation Guidance**:
  - Pattern: Run tests with coverage enabled (e.g., `pytest --cov --cov-branch`), parse coverage output, compare to thresholds
  - Spec validation rules: Thresholds are exactly 80% code and 90% branch. Below either -> add tests. At or above both -> report success.
  - Known pitfalls: Branch coverage is often harder to achieve than line coverage. The skill should identify uncovered branches specifically.

### T23-05 - Create code-integration-tests SKILL.md structure

- **Description**: Replace the stub code-integration-tests SKILL.md with the full skill file. Write the YAML frontmatter, input contract table, execution sequence, and output format sections.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] SKILL.md has valid YAML frontmatter with name `code-integration-tests` and description matching FR-006
  - [x] Input contract table lists all 8 inputs from FR-017
  - [x] Execution sequence follows FR-018
  - [x] Output format matches FR-019
  - [x] Common contract reference to `.github/skills/CODER-SKILL-CONTRACT.md` is included
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Same structure as code-env-setup SKILL.md (T22-01)
  - Files to modify: `.github/skills/code-integration-tests/SKILL.md`

### T23-06 - Write integration test generation logic

- **Description**: Write the code-integration-tests skill instructions for generating integration tests that verify component boundaries, test against real or mocked external dependencies, and include data setup/teardown.
- **Spec refs**: FR-031, FR-032
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL write tests across module boundaries within the WP's scope (FR-031.1)
  - [x] The skill SHALL write tests against real database/storage if the WP sets up data persistence (FR-031.2)
  - [x] The skill SHALL write tests against external API mocks using contract definitions for mock responses (FR-031.3)
  - [x] The skill SHALL include data setup and teardown for each test (FR-031.4)
  - [x] For external dependencies, the skill SHALL use contract files to generate mock responses that match exact schemas (FR-032.1)
  - [x] The skill SHALL test timeout, retry, and error handling paths (FR-032.2)
  - [x] The skill SHALL verify integration points match the API contract schemas (FR-032.3)
- **Test requirements**: BDD
- **Depends on**: T23-05
- **Implementation Guidance**:
  - Pattern: For each component boundary in the WP, create a test that exercises the real integration path with mock external dependencies
  - Known pitfalls: Integration tests must use contract-defined mock responses, not arbitrary test data. This ensures mocks match what the real service would return.

### T23-07 - Write integration test execution and reporting

- **Description**: Write the code-integration-tests skill instructions for running all integration tests after writing them and reporting results to the coordinator.
- **Spec refs**: FR-033
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL run all integration tests after writing them and report results (FR-033)
  - [x] Results SHALL include pass/fail counts and any error details
  - [x] Failed tests SHALL be reported to the coordinator for debug skill dispatch
- **Test requirements**: none
- **Depends on**: T23-06
- **Implementation Guidance**:
  - Pattern: Run integration tests in a separate test suite (e.g., `pytest tests/integration/`), report results in the FR-019 output format
  - Known pitfalls: Integration tests may require setup (databases, services). If prerequisites are missing, report clearly rather than silently skipping.

### T23-08 - Integration verification of both test skills with coordinator

- **Description**: Verify that both code-unit-tests and code-integration-tests SKILL.md files are correctly discovered by the coordinator's glob pattern, their contracts match, and their test result reporting matches what the coordinator expects for debug dispatch decisions.
- **Spec refs**: FR-005, FR-010, FR-017, FR-019
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Both skills are discovered by `file_search` with glob `.github/skills/code-*/SKILL.md`
  - [x] Input contract fields match the coordinator's dispatch template from Section 8.2
  - [x] Output contract fields match the coordinator's expected result format -- specifically test_results.fail_count drives debug dispatch (FR-010)
  - [x] No files contain em dashes, smart quotes, or curly apostrophes
- **Test requirements**: none
- **Depends on**: T23-07
- **Implementation Guidance**:
  - Pattern: Cross-reference coordinator debug logic (WP21 T21-07) to confirm it reads test_results.fail_count from skill output
  - Use encoding compliance check from T20-06

## Implementation Notes

- Both skills are markdown SKILL.md files containing instructions for the AI subagent, not executable code.
- Decision 3 from Section 9.4: Separate test skills from implementation. Testing in a fresh context window eliminates implementation bias.
- The unit test skill's BDD approach means tests derive from spec scenarios, not from reading source code. This is a deliberate design choice.
- Coverage thresholds (80% code, 90% branch) are enforced by the unit test skill. If not met, the skill self-iterates by adding more tests before reporting.
- Integration test mock responses must match contract schemas -- this is contract-first testing, not arbitrary mock data.
- The coordinator's debug dispatch decision depends on test_results.fail_count from these skills' output. The format must be consistent.

## Parallel Opportunities

- T23-01 (unit-tests structure) and T23-05 (integration-tests structure) can run in parallel [P]
- T23-03 (test validity rules) can run in parallel with T23-02 (test generation logic) [P]
- All other tasks are sequential within their skill

## Risks & Mitigations

- **Risk**: BDD-derived tests may not achieve 90% branch coverage for implementation-specific branches. **Mitigation**: The skill adds targeted tests for uncovered branches as a second pass.
- **Risk**: Integration test prerequisites (databases, services) may not be available in all environments. **Mitigation**: Skill reports missing prerequisites clearly; coordinator escalates if needed.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T14:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T14:10:00Z - coder - T23-01 - completed - code-unit-tests SKILL.md structure with frontmatter, input contract, execution sequence, output format
- 2026-04-05T14:15:00Z - coder - T23-02 - completed - BDD-derived test generation logic with scenario mapping
- 2026-04-05T14:18:00Z - coder - T23-03 - completed - Test validity rules with forbidden patterns and examples
- 2026-04-05T14:22:00Z - coder - T23-04 - completed - Test execution and coverage threshold enforcement (80% code, 90% branch)
- 2026-04-05T14:25:00Z - coder - T23-05 - completed - code-integration-tests SKILL.md structure with frontmatter, input contract, execution sequence, output format
- 2026-04-05T14:30:00Z - coder - T23-06 - completed - Integration test generation logic with boundary identification and contract-based mocking
- 2026-04-05T14:33:00Z - coder - T23-07 - completed - Integration test execution and reporting with prerequisite handling
- 2026-04-05T14:35:00Z - coder - T23-08 - completed - Integration verification: glob discovery, contract compliance, encoding compliance
- 2026-04-05T14:36:00Z - coder - lane=for_review - All tasks complete, tests passing, coverage met
- 2026-04-05T15:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (2 WARNs)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T15:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (WARN), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (PASS), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 8 tasks have acceptance criteria checked off
- [PASS] Activity Log: Consistent lane=planned -> lane=doing -> lane=for_review transitions with per-task entries
- [WARN] Commit granularity: All 8 tasks committed in a single commit (1974e51) instead of one commit per task (FR-016)
- [PASS] Encoding: No prohibited Unicode characters found

### Review Feedback

> No FAIL findings. No FB-XX items to address.

### Warnings
- [WARN] Commit granularity: All 8 tasks (T23-01 through T23-08) were committed in a single commit `1974e51` rather than individual commits per task as required by FR-016. (Process Compliance PROC-003)
- [WARN] Structural asymmetry: The integration test skill includes a "Handle Missing Prerequisites" section (Step 5) for infrastructure availability, but the unit test skill has no equivalent section for missing test tooling. Minor consistency gap. (review-quality QUAL-007)

### Cross-Correlation Notes
- No cross-correlation findings. No duplicates, conflicts, or systemic patterns detected.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 17 | 0 | 0 |
| review-security | 0 | 0 | 0 |
| review-quality | 6 | 1 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 4 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 3 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **33** | **2** | **0** |
