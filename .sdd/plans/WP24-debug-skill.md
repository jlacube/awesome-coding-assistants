---
lane: done
---

# WP24 - Debug Skill

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/004-coder-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP20, WP21 |
| Goal | Implement the code-debug SKILL.md that diagnoses test failures, fixes source or test code, re-runs tests, and reports results including regression detection |
| Status | Complete |
| Independent Test | Introduce a deliberate bug in implementation code. Run the Coder through tests. Verify: code-debug diagnoses the root cause, fixes the source code, re-runs all tests, and reports previously-failing tests now pass. Also verify: debug does not delete tests, weaken assertions, or add broad exception handlers |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP24-debug-skill.md` |

## Objective

Implement the conditional debug skill in the Coder V2 pipeline: `code-debug` (Phase 5). This skill is dispatched only when unit or integration tests fail. It reads failing test output, diagnoses root causes, fixes source code (preferring source fixes over test fixes), re-runs all tests to verify fixes and detect regressions, and reports results. The skill has strict safety constraints: it shall not delete tests, weaken assertions, add broad exception handlers, or modify contract files. The coordinator dispatches this skill up to 3 times before escalating to the human.

## Spec References

FR-034, FR-035, FR-036, FR-037 (code-debug), FR-017, FR-018, FR-019 (common skill contract), Section 4.7 (Debug Skill), Section 6.2 (Debug Flow), Section 8.3 (Debug Skill Prompt Template), Section 9.4 Decision 4

## Tasks

### T24-01 - Create code-debug SKILL.md structure

- **Description**: Replace the stub code-debug SKILL.md with the full skill file. Write the YAML frontmatter, input contract table (per CODER-SKILL-CONTRACT.md with additional debug-specific inputs), execution sequence, and output format sections.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md has valid YAML frontmatter with name `code-debug` and description matching FR-006
  - [x] Input contract includes the standard 8 inputs from FR-017 plus debug-specific inputs: failing test output, source file list, and debug attempt number
  - [x] The debug skill prompt template from Section 8.3 is incorporated into the skill's execution instructions
  - [x] Output format matches FR-019 with additional debug fields: previously-failing tests now passing, tests still failing, new regressions
  - [x] Common contract reference to `.github/skills/CODER-SKILL-CONTRACT.md` is included
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Same base structure as other skills, plus debug-specific inputs from Section 8.3
  - Files to modify: `.github/skills/code-debug/SKILL.md`
  - Known pitfalls: The debug skill has additional inputs beyond the standard 8 (failing test output, attempt number). These must be documented in the input contract.

### T24-02 - Write failure diagnosis logic

- **Description**: Write the code-debug skill instructions for reading failing test output, reading source code, reading contract files and spec sections, and diagnosing root causes of failures.
- **Spec refs**: FR-034
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL read failing test output (test names, error messages, stack traces) (FR-034.1)
  - [x] The skill SHALL read the corresponding source code (FR-034.2)
  - [x] The skill SHALL read the relevant contract files and spec sections (FR-034.3)
  - [x] The skill SHALL diagnose the root cause of each failure (FR-034.4)
  - [x] Given a unit test fails with a type mismatch, the debug skill diagnoses the mismatch (BDD Scenario 3)
- **Test requirements**: BDD
- **Depends on**: T24-01
- **Implementation Guidance**:
  - Pattern: Structured diagnosis: (1) read test error, (2) locate relevant source code, (3) compare against contract, (4) identify discrepancy
  - Known pitfalls: Multiple test failures may share a root cause. The skill should identify shared roots rather than fixing symptoms individually.

### T24-03 - Write fix prioritization and safety constraints

- **Description**: Write the code-debug skill rules for prioritizing source code fixes over test fixes, and the strict safety constraints preventing test deletion, assertion weakening, broad exception handling, and contract modification.
- **Spec refs**: FR-035, FR-036
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL prioritize source code fixes over test code fixes (FR-035)
  - [x] A test should only be modified if it genuinely tests the wrong behavior (not matching the spec) (FR-035)
  - [x] If the test correctly reflects the spec but the code is wrong, the code SHALL be fixed (FR-035)
  - [x] The skill SHALL NOT delete or skip failing tests (FR-036.1)
  - [x] The skill SHALL NOT weaken assertions to make tests pass (FR-036.2)
  - [x] The skill SHALL NOT add broad exception handlers to suppress errors (FR-036.3)
  - [x] The skill SHALL NOT modify contract files (FR-036.4)
- **Test requirements**: BDD
- **Depends on**: T24-02
- **Implementation Guidance**:
  - Pattern: Decision tree: (1) Is the test correct per spec? If yes -> fix source. If no -> fix test. Always cite the spec FR that justifies the choice.
  - Known pitfalls: The temptation to weaken assertions (e.g., changing `assertEqual` to `assertIn`) is a common anti-pattern. The skill must explicitly guard against this.
  - Error handling: Decision 5 from Section 9.4: Contract files are read-only. If the contract appears wrong, escalate to the Planner/Spec Architect.

### T24-04 - Write re-run verification and regression detection

- **Description**: Write the code-debug skill instructions for re-running all tests (unit + integration) after fixes and reporting detailed results including previously-failing tests that now pass, tests still failing, and new regressions.
- **Spec refs**: FR-037
- **Parallel**: No
- **Acceptance criteria**:
  - [x] After fixes, the skill SHALL re-run all tests (unit + integration) (FR-037)
  - [x] The skill SHALL report: previously failing tests that now pass (FR-037.1)
  - [x] The skill SHALL report: tests still failing with diagnosis (FR-037.2)
  - [x] The skill SHALL report: new failures introduced by fixes (regressions) (FR-037.3)
  - [x] Given the debug skill fixes a type mismatch, re-runs all tests successfully (BDD Scenario 3)
- **Test requirements**: BDD
- **Depends on**: T24-03
- **Implementation Guidance**:
  - Pattern: Capture test result diff between pre-fix and post-fix runs. Categorize changes: (1) was-failing-now-passing, (2) still-failing, (3) was-passing-now-failing (regression)
  - Known pitfalls: Regressions are critical -- the skill must re-run ALL tests, not just previously-failing ones, to catch regressions
  - Spec validation rules: The result report must allow the coordinator to determine if another debug attempt is needed (per FR-010.3)

### T24-05 - Write escalation reporting format

- **Description**: Write the code-debug skill reporting format for when it cannot diagnose a failure, providing full context for human escalation by the coordinator.
- **Spec refs**: FR-034 (error: cannot diagnose), FR-010.4
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] If the skill cannot diagnose a failure, it SHALL report to the coordinator with full context for human escalation (FR-034 error)
  - [x] The escalation report SHALL include: failing test names, error messages, stack traces, relevant source files, contract references, spec refs, and attempted fixes
  - [x] The report format SHALL allow the coordinator to present all context to the human (FR-010.4)
- **Test requirements**: none
- **Depends on**: T24-01
- **Implementation Guidance**:
  - Pattern: Structured escalation report with sections for each piece of context. Use the FR-019 output format with status=failure and detailed failure_reason.

### T24-06 - Integration verification with coordinator

- **Description**: Verify that code-debug SKILL.md is correctly discovered by the coordinator's glob pattern, its input contract matches the debug prompt template from Section 8.3, and its output contract matches the coordinator's debug retry decision logic.
- **Spec refs**: FR-005, FR-010, FR-017, FR-019
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill is discovered by `file_search` with glob `.github/skills/code-*/SKILL.md`
  - [x] Input contract fields match the debug prompt template from Section 8.3
  - [x] Output format enables the coordinator to determine: all-pass (done), still-failing (retry), or cannot-diagnose (escalate)
  - [x] No files contain em dashes, smart quotes, or curly apostrophes
- **Test requirements**: none
- **Depends on**: T24-05
- **Implementation Guidance**:
  - Pattern: Cross-reference coordinator debug dispatch logic (WP21 T21-07) with skill input/output contracts
  - Use encoding compliance check from T20-06

## Implementation Notes

- The debug skill is a markdown SKILL.md file containing instructions for the AI subagent, not executable code.
- Decision 4 from Section 9.4: Debug skill with 3-attempt budget. Most test failures are minor bugs fixable by the agent. The 3-attempt cap prevents infinite loops.
- The coordinator controls the retry loop (FR-010), not the debug skill. The skill runs once per invocation and reports results. The coordinator decides whether to retry.
- The debug prompt template (Section 8.3) includes the attempt counter ("Debug attempt: N of 3"), which helps the skill understand urgency and escalation context.
- Fix prioritization (FR-035) is critical: the skill must analyze whether the test or the code is wrong relative to the spec, not just make the test pass by any means.

## Parallel Opportunities

- T24-05 (escalation reporting) can run in parallel with T24-02 through T24-04 [P]
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: Debug skill introduces regressions when fixing failures. **Mitigation**: Mandatory re-run of ALL tests (unit + integration) after every fix. Regression detection in result reporting.
- **Risk**: Debug skill misidentifies the root cause and "fixes" the wrong code. **Mitigation**: Structured diagnosis requires citing the spec FR and contract reference that justifies the fix.
- **Risk**: 3-attempt budget may be insufficient for complex failures. **Mitigation**: Decision 4 rationale: 3 attempts balances autonomy with escalation. Complex failures should go to the human.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T19:30:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (PASS), review-architecture (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 30+ acceptance criteria checked [x] across 6 tasks
- [PASS] Activity Log: Consistent lane transitions (planned -> doing -> for_review)
- [PASS] Commit granularity: One commit per task (T24-01 through T24-06), plus submission commit
- [PASS] Encoding: No prohibited Unicode characters found in SKILL.md or WP file

### Review Feedback

No FB-XX items. All spec requirements fully satisfied.

### Warnings

No warnings.

### Cross-Correlation Notes

No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 18 | 0 | 0 |
| review-security | 4 | 0 | 0 |
| review-quality | 3 | 0 | 0 |
| review-architecture | 2 | 0 | 0 |
| **Total** | **31** | **0** | **0** |

#### Spec Adherence Detail
- FR-017: Input contract has 8 standard + 3 debug-specific inputs. PASS.
- FR-018: 5-step execution sequence matches common contract. PASS.
- FR-019: Standard output fields + debug-specific fields (tests_fixed, tests_still_failing, regressions). PASS.
- FR-034.1: Step 1 reads test output (names, errors, stack traces). PASS.
- FR-034.2: Step 1b reads corresponding source code. PASS.
- FR-034.3: Step 1c reads contract files and spec sections. PASS.
- FR-034.4: Step 2 diagnoses root causes with decision tree. PASS.
- FR-035: Step 3a prioritizes source fixes with spec-based decision flow. PASS.
- FR-036.1: Safety constraints forbid deleting/skipping tests with specific examples. PASS.
- FR-036.2: Forbidden assertion-weakening patterns table (6 patterns). PASS.
- FR-036.3: Broad exception handler prohibition with examples. PASS.
- FR-036.4: Contract file modification prohibition with path reference. PASS.
- FR-037: Step 4 re-runs ALL tests (unit + integration). PASS.
- FR-037.1: Fixed tests categorization (was-failing, now-passes). PASS.
- FR-037.2: Still-failing tests with diagnosis. PASS.
- FR-037.3: Regression detection (was-passing, now-fails). PASS.
- Section 8.3: Prompt template inputs match SKILL.md input contract. PASS.
- Section 9.4 Decision 4: 3-attempt budget correctly documented as coordinator-controlled. PASS.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T12:01:00Z - coder - T24-01 - completed - SKILL.md structure with frontmatter, input/output contracts, execution sequence
- 2026-04-05T12:02:00Z - coder - T24-02 - completed - Failure diagnosis logic with categorization, source code reading, diagnosis decision tree
- 2026-04-05T12:03:00Z - coder - T24-03 - completed - Fix prioritization and safety constraints (FR-035, FR-036)
- 2026-04-05T12:04:00Z - coder - T24-04 - completed - Re-run verification and regression detection (FR-037)
- 2026-04-05T12:05:00Z - coder - T24-05 - completed - Escalation reporting format with full context for human review
- 2026-04-05T12:06:00Z - coder - T24-06 - completed - Integration verification: glob discovery, prompt template match, encoding compliance
- 2026-04-05T12:07:00Z - coder - lane=for_review - All tasks complete, all acceptance criteria met
- 2026-04-05T19:30:00Z - review-coordinator - lane=done - Verdict: Approved
