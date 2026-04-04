---
lane: to_do
review_status: has_feedback
---

# WP06 - P2 Review Skills (review-tests, review-architecture)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P2 |
| Lane | to_do |
| Depends on | WP02 |
| Goal | Create two P2 review skills: review-tests (test quality evaluation) and review-architecture (architecture adherence), extending the coordinator's review coverage beyond MVP dimensions |
| Status | Complete |
| Independent Test | Install both P2 skills alongside P1 skills. Invoke the coordinator on a WP with test quality issues (vacuous test, missing BDD scenario) and architecture issues (scope creep, wrong technology). Verify: coordinator dispatches all 5 skills, findings files for review-tests and review-architecture contain appropriate FAIL/WARN entries |
| Parallelisable | Yes (with WP07; internal tasks for each skill are also independent) |
| Prompt | `.sdd/plans/WP06-p2-skills.md` |

## Objective

Create two P2 review skills that add test quality and architecture adherence review dimensions. These skills follow the same contract (FR-025-029) and file format (Section 7.1) established by the P1 skills in WP03-05. They are dispatched by the coordinator in positions 4 and 5 of the canonical order (FR-004). These are post-MVP enhancements that deepen review coverage without changing the coordinator.

## Spec References

- Section 4.2 (FR-025 to FR-029) - Common skill contract
- Section 4.6 (FR-040 to FR-041) - Test quality skill
- Section 4.7 (FR-042 to FR-043) - Architecture skill
- Section 7.1 (Skill Findings File format)
- Section 7.5 (Skill File metadata)
- Section 11.2 (BDD scenarios, test and architecture references)

## Tasks

### T06-01 - Create review-tests SKILL.md with frontmatter and purpose

- **Description**: Create the directory `.github/skills/review-tests/` and file `SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: Yes (independent of T06-04 through T06-06)
- **Acceptance criteria**:
  - [x] Directory `.github/skills/review-tests/` exists
  - [x] File `.github/skills/review-tests/SKILL.md` exists
  - [x] YAML frontmatter `name` is `review-tests`
  - [x] YAML frontmatter `description` explains: evaluates test validity, coverage, BDD matching, edge cases, structure, and error path testing
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the same SKILL.md pattern established in WP03-05:
    ```yaml
    ---
    name: review-tests
    description: "Test quality review skill. Evaluates test validity, coverage thresholds, BDD scenario matching, edge cases, test structure, and error path testing."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T06-02 - Write test quality checklist

- **Description**: Write the 6-dimension test quality checklist from FR-040.
- **Spec refs**: FR-040 (6 test quality dimensions)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Test Validity checks: tests can fail, flag `assert True`, empty bodies, no assertions, mocking away entire subject (vacuous tests)
  - [x] Coverage Thresholds checks: code coverage >= 80%, branch coverage >= 90% for WP files. Flag files below threshold. Flag `# pragma: no cover` without justification
  - [x] BDD Scenario Matching checks: every acceptance scenario from spec Sections 5 and 11.2 mapped to this WP has a corresponding test
  - [x] Edge Case Coverage checks: error paths, boundary values, empty inputs, max inputs, concurrent access tested
  - [x] Test Structure checks: Arrange/Act/Assert (or Given/When/Then), isolated tests (no shared mutable state), descriptive test names
  - [x] Error Path Testing checks: every specified error response has at least one test exercising it
  - [x] Each dimension has at least 3 verifiable checklist items
- **Test requirements**: BDD (reference spec Section 11.2 test-related scenarios)
- **Depends on**: T06-01
- **Implementation Guidance**:
  - Vacuous test detection is the most critical check - tests that cannot fail provide false confidence
  - Coverage thresholds (80% code, 90% branch) match the values enforced by the Coder agent (Section 12 Constraints)
  - BDD Scenario Matching: the subagent reads spec Section 5 (User Stories, acceptance scenarios) and Section 11.2 (BDD scenarios), then checks which map to the current WP's FRs, then verifies test files exist for those scenarios
  - `# pragma: no cover` or equivalent exclusions are acceptable ONLY with documented justification in the code

### T06-03 - Write review-tests severity guidance and output format

- **Description**: Write severity rules per FR-041 and the output format matching Section 7.1.
- **Spec refs**: FR-041 (severity rules), FR-027 (findings format), FR-028 (read-only), FR-029 (N/A)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] FAIL: vacuous tests (`assert True`, empty bodies, mocking entire subject), coverage below threshold without justification, missing BDD scenario coverage
  - [x] WARN: test naming issues, minor structural concerns (shared setup that could be isolated)
  - [x] N/A with justification for dimensions not applicable to the WP
  - [x] Finding prefix is `TEST-` (e.g., `TEST-001`, `TEST-002`)
  - [x] Output format matches Section 7.1 with `files_reviewed` field
  - [x] Read-only constraint: "Do NOT modify any test files, source code, WP file, or spec file" (FR-028)
  - [x] Complete example findings file included
- **Test requirements**: BDD - reference test quality BDD scenarios
- **Depends on**: T06-02
- **Implementation Guidance**:
  - Example finding:
    ```markdown
    ### TEST-001 [FAIL]
    - **Checklist item**: Test Validity - Vacuous test
    - **Requirement**: FR-040 dimension 1
    - **File**: tests/test_users.py#L25-L28
    - **Description**: Test `test_user_creation` contains only `assert True`.
    - **Expected**: Test should assert specific behavior of the user creation function.
    - **Evidence**:
      ```python
      def test_user_creation():
          assert True
      ```
    ```

### T06-04 - Create review-architecture SKILL.md with frontmatter and purpose

- **Description**: Create the directory `.github/skills/review-architecture/` and file `SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: Yes (independent of T06-01 through T06-03)
- **Acceptance criteria**:
  - [x] Directory `.github/skills/review-architecture/` exists
  - [x] File `.github/skills/review-architecture/SKILL.md` exists
  - [x] YAML frontmatter `name` is `review-architecture`
  - [x] YAML frontmatter `description` explains: evaluates architecture adherence, component design, tech stack compliance, SOLID principles, scope discipline
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Same SKILL.md pattern:
    ```yaml
    ---
    name: review-architecture
    description: "Architecture adherence review skill. Evaluates component design, tech stack compliance, directory structure, SOLID principles, dependency direction, and scope discipline."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T06-05 - Write architecture adherence checklist

- **Description**: Write the 8-dimension architecture checklist from FR-042.
- **Spec refs**: FR-042 (8 architecture dimensions)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Component Adherence: implemented components match spec Section 9.1
  - [x] Technology Stack Compliance: technologies match spec Section 9.2, no unauthorized substitutions
  - [x] Directory Structure Compliance: file locations match spec Section 9.3
  - [x] Key Design Decisions: architectural decisions from spec Section 9.4 are honored
  - [x] Separation of Concerns: each module has single clear responsibility, no god objects
  - [x] SOLID Principles: SRP especially. Flag classes/modules with multiple unrelated responsibilities
  - [x] Dependency Direction: high-level modules do not depend on low-level implementation details
  - [x] Scope Discipline: no code outside WP tasks, no files modified beyond WP scope, no unspecified features/abstractions/utilities
  - [x] Each dimension has at least 3 verifiable checklist items
- **Test requirements**: BDD (reference architecture-related scenarios)
- **Depends on**: T06-04
- **Implementation Guidance**:
  - Scope Discipline (dimension 8) is the most critical and most commonly violated check
  - The subagent reads the WP's task descriptions to determine the declared scope, then verifies no files outside that scope were modified
  - Use `get_changed_files` or `git diff` to identify files modified by the WP implementation
  - Technology stack compliance: compare technologies used (imports, dependencies) against spec Section 9.2 table
  - Dependency direction: for each import/require statement, verify it points from higher-level to lower-level modules per the architecture

### T06-06 - Write review-architecture severity guidance and output format

- **Description**: Write severity rules per FR-043 and the output format matching Section 7.1.
- **Spec refs**: FR-043 (severity rules), FR-027 (findings format), FR-028 (read-only), FR-029 (N/A)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] FAIL: scope creep (unspecified code), technology stack violations, component design violations
  - [x] WARN: SRP concerns, minor structural deviations, dependency direction suggestions
  - [x] N/A with justification for dimensions not applicable
  - [x] Finding prefix is `ARCH-` (e.g., `ARCH-001`, `ARCH-002`)
  - [x] Output format matches Section 7.1 with `files_reviewed` field
  - [x] Read-only constraint (FR-028) and complete example included
- **Test requirements**: BDD - reference architecture BDD scenarios
- **Depends on**: T06-05
- **Implementation Guidance**:
  - Example finding:
    ```markdown
    ### ARCH-001 [FAIL]
    - **Checklist item**: Scope Discipline - Unspecified code
    - **Requirement**: FR-042 dimension 8
    - **File**: src/utils/cache.py
    - **Description**: Cache utility module was created but is not specified in any WP task.
    - **Expected**: Only implement code required by WP tasks. Remove unspecified utilities.
    - **Evidence**: No WP task references a cache module. The spec does not mention caching.
    ```

### T06-07 - Integration verification of P2 skills with coordinator

- **Description**: Verify both P2 skills integrate correctly with the coordinator by checking the end-to-end flow: dynamic discovery picks up the new skills, dispatch ordering is correct (review-tests at position 4, review-architecture at position 5), findings files are created with correct prefixes, and aggregate verdict includes P2 dimensions.
- **Spec refs**: FR-003 (discovery), FR-004 (dispatch order), FR-010 (aggregation)
- **Parallel**: No (final task)
- **Acceptance criteria**:
  - [x] Coordinator discovers review-tests and review-architecture via glob scan
  - [x] Dispatch order is: review-spec, review-security, review-quality, review-tests, review-architecture (positions 4 and 5)
  - [x] Findings files are named: `review-tests-findings.md` and `review-architecture-findings.md`
  - [x] Finding prefixes are `TEST-` and `ARCH-` respectively
  - [x] Aggregate verdict includes P2 skill findings in the statistics table
  - [x] P2 skill findings participate in cross-correlation with P1 skill findings
- **Test requirements**: BDD - "Dynamic skill discovery" scenario with 5 skills
- **Depends on**: T06-03, T06-06 (both skills must be complete)
- **Implementation Guidance**:
  - This is a verification task, not a code change. The Coder should:
    1. Install all 5 skills (3 P1 + 2 P2) in `.github/skills/review-*/`
    2. Invoke the coordinator on a test WP
    3. Verify the 5-skill dispatch, findings file creation, and aggregated report
  - If discovery or dispatch fails, the issue is in the coordinator (WP02) or the skill file format, not in this WP's code

## Implementation Notes

- This WP produces TWO files: `.github/skills/review-tests/SKILL.md` and `.github/skills/review-architecture/SKILL.md`
- Each file follows the same structure established by P1 skills: Purpose -> Checklist -> Severity Rules -> Output Format
- Both skills are independent of each other - their tasks can be interleaved or worked in any order
- Target size: 120-180 lines each. These skills have fewer checklist items than security (6 dimensions and 8 dimensions vs 14 categories)
- P2 directory creation (`.github/skills/review-tests/`, `.github/skills/review-architecture/`) is handled in this WP, not in WP01

## Parallel Opportunities

- T06-01 through T06-03 (review-tests) and T06-04 through T06-06 (review-architecture) are fully parallel tracks
- T06-07 depends on both tracks being complete

## Risks & Mitigations

- **Risk**: Coverage threshold checks (80/90) are not measurable by the subagent without tooling
  - Mitigation: The subagent can check if coverage configuration exists (pytest-cov, istanbul config), run coverage commands if available, or flag the absence of coverage tooling. The skill should instruct: "If coverage tooling is configured, run it and report thresholds. If not configured, flag as WARN."
- **Risk**: Scope discipline check produces false positives for utility functions needed by WP tasks
  - Mitigation: Include guidance that helper functions directly required by WP tasks are in scope. Only flag code with zero traceability to any WP task.

## Review

### Round 1

| Field | Value |
|-------|-------|
| Reviewer | 5. Review Coordinator |
| Date | 2026-04-05 |
| Verdict | Changes Required |
| Round | 1 |

#### Aggregated Statistics

| Skill | PASS | WARN | FAIL | N/A |
|-------|------|------|------|-----|
| review-spec | 21 | 0 | 0 | 7 |
| review-security | 3 | 1 | 1 | 14 |
| review-quality | 4 | 0 | 0 | 4 |
| review-tests | 0 | 0 | 0 | 6 |
| review-architecture | 16 | 0 | 0 | 4 |
| review-performance | 1 | 0 | 0 | 7 |
| review-docs | 18 | 1 | 0 | 14 |
| review-deps | 0 | 0 | 0 | 6 |
| **Total** | **63** | **2** | **1** | **62** |

#### FAIL Findings

##### FB-01: SEC-001 - NFR-004 violation: review-tests instructs code execution
- [ ] Resolved
- **Severity**: FAIL
- **Source**: review-security SEC-001
- **Requirement**: NFR-004 ("Skills SHALL NOT execute any discovered code. Review is static analysis only.")
- **File**: `.github/skills/review-tests/SKILL.md` line 51
- **Description**: The Coverage Thresholds dimension instructs the subagent to "run the coverage tool (e.g., `pytest --cov --cov-branch`) and report actual thresholds." Executing `pytest` runs the project's test suite and transitively the project's source code, violating the spec's static-analysis-only constraint.
- **Required fix**: Replace the code execution instruction with a static analysis approach. The skill should instruct the subagent to: (1) check if coverage configuration exists (pytest-cov in pyproject.toml, .coveragerc, istanbul config, etc.), (2) read existing coverage reports if available (htmlcov/, coverage.xml, .coverage), (3) flag if no coverage tooling is configured. The subagent must never execute test runners or coverage tools.

#### WARN Findings (Acknowledged)

##### SEC-002: Neither P2 skill includes NFR-005 secret non-reproduction constraint
- **Severity**: WARN
- **Source**: review-security SEC-002
- **Requirement**: NFR-005
- **Description**: Neither review-tests nor review-architecture includes an explicit constraint against reproducing secret values in evidence snippets. This is defense-in-depth -- these skills are unlikely to encounter secrets in their review domains, but the constraint is present in review-security and would be beneficial for consistency.
- **Action**: Recommended but not blocking. Coder may address at their discretion.

##### DOC-WARN: 3 of 6 standard doc files missing
- **Severity**: WARN
- **Source**: review-docs
- **Description**: api-reference.md, configuration-guide.md, deployment-guide.md do not exist. These cover domains not applicable to this markdown-only framework project.
- **Action**: Accepted. Recurring WARN across all WPs -- project type justifies absence.

#### Cross-Correlation Notes

- SEC-001 is unique to review-security. No other skill flagged the coverage execution instruction, confirming review-security is the correct skill to catch NFR-004 violations.
- No duplicate findings across skills. All findings are independent.
- Process WARNs (single commit per WP, missing N/A doc files) are consistent with prior WP reviews and accepted.

#### Findings Directory

All individual skill findings are in `.sdd/reviews/WP06-p2-skills/`:
- `review-spec-findings.md`
- `review-security-findings.md`
- `review-quality-findings.md`
- `review-tests-findings.md`
- `review-architecture-findings.md`
- `review-performance-findings.md`
- `review-docs-findings.md`
- `review-deps-findings.md`

## Activity Log

- 2026-04-04T11:35:00Z - planner - lane=planned - Work package created
- 2026-04-04T16:00:00Z - coder - lane=doing - Starting WP06 implementation
- 2026-04-04T16:30:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T00:00:00Z - reviewer - lane=to_do - Round 1 review: Changes Required (1 FAIL: SEC-001 NFR-004 violation)
