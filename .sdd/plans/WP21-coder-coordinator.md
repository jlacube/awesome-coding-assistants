---
lane: doing
---

# WP21 - Coder Coordinator

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/004-coder-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP20 |
| Goal | Rewrite coder.agent.md from monolithic single-pass implementation to a lightweight coordinator that dispatches 5 sequential coding skills with contract-first enforcement, debug retry logic, and no self-review |
| Status | Not Started |
| Independent Test | Invoke the Coder with a WP. Verify: it reads WP + contracts + spec, validates dependencies, discovers 5 coding skills, dispatches them sequentially, handles debug retries, sets lane to for_review, commits per task, and hands off to Reviewer without self-review |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP21-coder-coordinator.md` |

## Objective

Rewrite `.github/agents/coder.agent.md` from the current monolithic implementation agent into a lightweight coordinator that dispatches 5 sequential coding skills. The coordinator handles the full WP lifecycle: WP selection and validation, artifact chain loading, contract file verification, patterns consumption, dynamic skill discovery, sequential skill dispatch, conditional debug with 3-attempt retry, task state tracking, coverage verification, commit policy enforcement, and handoff to the Reviewer. The coordinator writes no implementation code itself -- it orchestrates skills. Self-review is explicitly removed.

## Spec References

FR-001 through FR-016, Section 6.1 (Full WP Implementation Flow), Section 6.2 (Debug Flow), Section 7.1 (WP File State Transitions), Section 8.1-8.4 (Interface Design), Section 9.1 (System Design), Section 9.4 (Key Design Decisions)

## Tasks

### T21-01 - Write WP selection and user interaction

- **Description**: Write the coordinator logic for listing `.sdd/plans/WP*.md` files and selecting the WP to implement. If a WP is provided as an argument, use it directly. If not, present the list to the user via `vscode_askQuestions`.
- **Spec refs**: FR-001
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL list all `.sdd/plans/WP*.md` files and select the specified WP or present the list to the user via `vscode_askQuestions` (FR-001)
  - [x] If no WP files exist, the coordinator SHALL halt and inform the user (FR-001 error)
  - [x] Given a user selects WP03 from a list of 5 WPs, the coordinator proceeds with WP03
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Use `list_dir` to scan `.sdd/plans/` for `WP*.md` files, then `vscode_askQuestions` for selection
  - Error handling: Empty directory -> halt with "No WP files found in .sdd/plans/. Run the Planner first."
  - Files to modify: `.github/agents/coder.agent.md`

### T21-02 - Write artifact chain loading

- **Description**: Write the coordinator logic for reading the full context chain before dispatching skills: the selected WP file, `.sdd/plans/README.md` for sequencing context, the spec sections referenced by the WP, contract files in `.sdd/plans/contracts/<WP-slug>/`, and `AGENTS.md` at workspace root.
- **Spec refs**: FR-002
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL read the selected WP file in full (FR-002.1)
  - [x] The coordinator SHALL read `.sdd/plans/README.md` for sequencing context and dependency status (FR-002.2)
  - [x] The coordinator SHALL read the spec sections referenced in the WP (FR-002.3)
  - [x] The coordinator SHALL read contract files in `.sdd/plans/contracts/<WP-slug>/` (FR-002.4)
  - [x] The coordinator SHALL read `AGENTS.md` at workspace root if it exists (FR-002.5)
  - [x] If a dependency WP has `lane` not equal to `done`, the coordinator SHALL halt and recommend completing the dependency first (FR-002 error)
- **Test requirements**: BDD
- **Depends on**: T21-01
- **Implementation Guidance**:
  - Pattern: Extract WP slug from filename (e.g., `WP03-review-spec` -> slug is `review-spec`), then construct contracts path `.sdd/plans/contracts/review-spec/`
  - Error handling: Dependency check reads each dependency WP's YAML frontmatter `lane:` field. If not `done`, halt.
  - Known pitfalls: AGENTS.md is optional -- do not fail if it does not exist.

### T21-03 - Write contract file validation

- **Description**: Write the coordinator logic for verifying that all contract files referenced by the WP's tasks exist and contain valid syntax.
- **Spec refs**: FR-003
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL verify all contract files referenced by the WP's tasks exist and contain valid syntax (FR-003)
  - [x] If any contract file is missing, the coordinator SHALL halt and recommend re-running the Planner (FR-003 error)
  - [x] Given a WP references `interfaces.ts` but it does not exist in the contracts directory, the coordinator halts with a recommendation to re-run the Planner
- **Test requirements**: BDD
- **Depends on**: T21-02
- **Implementation Guidance**:
  - Pattern: Parse task descriptions for contract file references, then verify each file exists using `read_file` or `list_dir`
  - Error handling: Missing file -> halt with "Contract file <path> referenced by task T<NN>-XX is missing. Re-run the Planner to generate contracts."

### T21-04 - Write patterns consumption

- **Description**: Write the coordinator logic for reading `.sdd/reviews/code-patterns.md` before dispatching skills and including active pattern summaries in each skill's prompt.
- **Spec refs**: FR-004
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL read `.sdd/reviews/code-patterns.md` if it exists before dispatching any skill (FR-004)
  - [x] Active pattern summaries SHALL be included in the prompt for each skill (FR-004)
  - [x] Skills SHALL avoid producing code that would trigger known patterns (FR-004)
  - [x] If `code-patterns.md` does not exist, the coordinator SHALL proceed without error
- **Test requirements**: none
- **Depends on**: T21-02
- **Implementation Guidance**:
  - Pattern: Same approach as the Planner coordinator's patterns consumption (see `.github/agents/planner.agent.md`)
  - Files to read: `.sdd/reviews/code-patterns.md`

### T21-05 - Write dynamic skill discovery and ordering

- **Description**: Write the coordinator logic for discovering available coding skills by scanning `.github/skills/code-*/SKILL.md` and ordering them in the canonical sequence from FR-006.
- **Spec refs**: FR-005, FR-006
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL discover available coding skills by scanning for directories matching `.github/skills/code-*/SKILL.md` (FR-005)
  - [x] A sorted list of discovered skill names SHALL be produced (FR-005 postcondition)
  - [x] If zero skills are discovered, the coordinator SHALL halt and report no coding skills installed (FR-005 error)
  - [x] The coordinator SHALL dispatch skills in deterministic order: code-env-setup, code-implementation, code-unit-tests, code-integration-tests, code-debug (FR-006)
  - [x] Skills not present in the canonical list SHALL be dispatched after all known skills, in alphabetical order (FR-006)
  - [x] Given 5 coding skills plus a 6th `code-linting`, then 6 skills are dispatched with `code-linting` last (US-05)
- **Test requirements**: BDD
- **Depends on**: T21-03
- **Implementation Guidance**:
  - Pattern: Use `file_search` with glob `.github/skills/code-*/SKILL.md`, then sort by canonical order
  - Known pitfalls: The canonical order list is fixed in the coordinator. Unknown skills go after known ones, sorted alphabetically.
  - Spec validation rules: Canonical order is exactly: code-env-setup, code-implementation, code-unit-tests, code-integration-tests, code-debug

### T21-06 - Write skill dispatch via runSubagent

- **Description**: Write the coordinator logic for dispatching each discovered skill as a subagent invocation using `runSubagent`. Each invocation SHALL include the 8 inputs from FR-017 using the prompt template from Section 8.2. Skills execute sequentially, one at a time.
- **Spec refs**: FR-007, FR-008, FR-009
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each skill SHALL be dispatched as a subagent invocation using `runSubagent` (FR-007)
  - [x] Each invocation SHALL include: skill file path, WP file path, contracts directory, spec file path, active patterns, target language, target framework, and task list with acceptance criteria (FR-007)
  - [x] If a skill fails (environment setup or implementation), the coordinator SHALL halt the WP and report the error; it SHALL NOT proceed to later skills (FR-007 error)
  - [x] Skills SHALL execute sequentially, one at a time, blocking (FR-008)
  - [x] The coordinator SHALL NOT dispatch the next skill until the current skill completes (FR-008)
  - [x] Each skill SHALL read the current state of the codebase (files created or modified by prior skills) before executing (FR-009)
- **Test requirements**: BDD
- **Depends on**: T21-05
- **Implementation Guidance**:
  - Pattern: Use the prompt template from Section 8.2 verbatim for each skill dispatch
  - Known pitfalls: FR-009 context forwarding is automatic since each subagent reads the filesystem fresh
  - Error handling: Skill failure -> halt WP, do not dispatch remaining skills. Report failure with full context.

### T21-07 - Write conditional debug dispatch with retry logic

- **Description**: Write the coordinator logic for checking test results after unit and integration test skills, and conditionally dispatching the `code-debug` skill with up to 3 retry attempts if tests fail. Use the debug prompt template from Section 8.3.
- **Spec refs**: FR-010
- **Parallel**: No
- **Acceptance criteria**:
  - [x] After unit and integration test skills complete, the coordinator SHALL check test results (FR-010)
  - [x] If any tests fail, the coordinator SHALL dispatch `code-debug` with failing test output, relevant source files, contract files, and spec refs (FR-010.1)
  - [x] The debug skill SHALL diagnose failures, fix the code, and re-run tests (FR-010.2)
  - [x] If tests still fail after the debug skill, the coordinator SHALL retry up to 2 more times (max 3 debug attempts total) (FR-010.3)
  - [x] If tests still fail after 3 debug attempts, the coordinator SHALL escalate to the human with full error context (FR-010.4)
  - [x] If all tests pass after the test skills, the debug skill SHALL NOT be dispatched (FR-010)
  - [x] Given tests fail due to a fundamental issue, when code-debug fails after 3 attempts, then the coordinator escalates to the human and does not mark the WP as for_review (BDD Scenario 4)
- **Test requirements**: BDD
- **Depends on**: T21-06
- **Implementation Guidance**:
  - Pattern: Use the debug prompt template from Section 8.3, incrementing the attempt counter (1 of 3, 2 of 3, 3 of 3)
  - Error handling: After 3 failures, escalate with full context: failing tests, source files, contracts, spec refs, all 3 debug attempt outputs
  - Known pitfalls: Must re-run ALL tests (unit + integration) after each debug fix, not just the failing ones. Debug skill may introduce regressions.

### T21-08 - Write task state tracking and WP lifecycle

- **Description**: Write the coordinator logic for updating WP lane state and tracking individual task progress using `manage_todo_list` and WP file updates.
- **Spec refs**: FR-011, FR-012, FR-013
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL update the WP file's `lane:` frontmatter to `doing` when implementation begins (FR-011)
  - [x] The coordinator SHALL track each task's progress using `manage_todo_list`, marking tasks in-progress when started and completed when acceptance criteria are met (FR-012)
  - [x] The coordinator SHALL update the WP file after each task completes: check off acceptance criteria and append an Activity Log entry (FR-013)
  - [x] Activity Log entries SHALL include task ID, status, and timestamp per Section 7.3
- **Test requirements**: none
- **Depends on**: T21-06
- **Implementation Guidance**:
  - Pattern: Use `replace_string_in_file` to update `lane:` in YAML frontmatter and check off `- [ ]` to `- [x]`
  - Spec validation rules: Activity Log entry format: `- <timestamp> - coder - <task_id> - <status> - <notes>`

### T21-09 - Write post-completion handoff and coverage verification

- **Description**: Write the coordinator logic for running a final coverage report, verifying thresholds (80% code, 90% branch), setting the WP's lane to `for_review`, and handing off to the Review Coordinator. Explicitly: NO self-review step occurs.
- **Spec refs**: FR-014, FR-015
- **Parallel**: No
- **Acceptance criteria**:
  - [x] After all tasks are complete and tests pass, the coordinator SHALL run a final coverage report and verify thresholds: 80% code coverage, 90% branch coverage (FR-014.1)
  - [x] The coordinator SHALL set the WP's `lane:` frontmatter to `for_review` (FR-014.2)
  - [x] The coordinator SHALL hand off to the Review Coordinator using the handoff prompt from Section 8.4 (FR-014.4)
  - [x] The coordinator SHALL NOT perform any self-assessment, self-review, or quality evaluation of the code (FR-015)
  - [x] Given a completed WP, when all tests pass, the WP goes directly to for_review without any review step (BDD Scenario 2)
- **Test requirements**: BDD
- **Depends on**: T21-07
- **Implementation Guidance**:
  - Pattern: Use the "Request Review" handoff template from Section 8.4 verbatim
  - Known pitfalls: FR-015 is a SHALL NOT -- there must be no self-review language in the coordinator. No quality assessment, no code review, no "verified implementation quality" statements.
  - Error handling: Coverage below thresholds -> re-dispatch test skills to add more tests (per BDD Scenario 8)

### T21-10 - Write commit policy and handoff prompts

- **Description**: Write the coordinator logic for per-task commits with explicit file listing, and the handoff prompt templates for requesting review and clarifying specification ambiguities.
- **Spec refs**: FR-016, Section 8.4
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each task SHALL be committed individually with `git add <explicit file list>` followed by `git commit -m "<type>(<scope>): <description> (WP<NN> T<NN>-XX)"` (FR-016)
  - [x] Commit types SHALL be one of: feat, fix, refactor, test, docs, chore (FR-016)
  - [x] Task ID SHALL always be included at the end of the commit message (FR-016)
  - [x] Files SHALL be listed explicitly in `git add` -- never `git add .` or `git add -A` (FR-016)
  - [x] The "Request Review" handoff template from Section 8.4 SHALL be included
  - [x] The "Clarify Specification" handoff template from Section 8.4 SHALL be included
- **Test requirements**: none
- **Depends on**: T21-09
- **Implementation Guidance**:
  - Pattern: Copy handoff templates from Section 8.4 verbatim
  - Known pitfalls: Never use `git add .` or `git add -A`. Always list files explicitly.
  - Spec validation rules: Commit message format is `<type>(<scope>): <description> (WP<NN> T<NN>-XX)` where type is one of the 6 allowed values

## Implementation Notes

- The coordinator is a single markdown file (`.github/agents/coder.agent.md`) that replaces the existing monolithic coder.
- All implementation artifacts are markdown instructions. No executable code is produced by this WP.
- The coordinator relies on `runSubagent` for skill dispatch and `manage_todo_list` for task tracking -- both are VS Code Copilot Chat tools.
- The debug retry logic (FR-010) is the most complex state management in the coordinator. Use a clear attempt counter pattern.
- Decision 1 from Section 9.4: Self-review is explicitly removed. The coordinator MUST NOT contain any self-assessment language.
- Decision 5 from Section 9.4: Contract files are read-only. The coordinator instructs skills to never modify contracts.

## Parallel Opportunities

- T21-04 (patterns consumption) can run in parallel with T21-03 (contract validation)
- T21-08 (task state tracking) can run in parallel with T21-07 (debug dispatch)
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: Coordinator logic exceeds context window limits for a single agent file. **Mitigation**: Use concise markdown; follow the same density as planner.agent.md which handles similar complexity.
- **Risk**: Debug retry state management becomes fragile. **Mitigation**: Use simple counter pattern with explicit max (3); escalate on any unexpected state.
- **Risk**: Handoff target names mismatch with other agents. **Mitigation**: Cross-reference existing agent handoff buttons before writing.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T14:00:00Z - coder - lane=doing - Starting implementation of coordinator rewrite
