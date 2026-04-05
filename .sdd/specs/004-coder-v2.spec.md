# Coder V2 - Skill-Based Contract-First Implementation -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `004-coder-v2`
> **Status**: Approved
> **Version**: 1.1

---

## 1. Overview

Replace the monolithic Coder agent (single-pass implementation with self-review) with a skill-based Coder coordinator that dispatches 5 sequential skills: environment setup, core implementation, unit tests, integration tests, and debugging on failure. The self-review phase is removed entirely -- the Reviewer agent is the dedicated quality gate. The Coder V2 implements contract-first: it reads language-specific contract files from `.sdd/plans/contracts/` (interfaces, data schemas, API contracts, state machines, error catalogs) and implements against them verbatim rather than interpreting prose requirements. Each skill runs in a fresh context window, enabling focused execution without context overload.

---

## 2. Goals & Success Criteria

- **SC-001**: Implementation code aligns with contract files -- function signatures, field names, types, and error codes in the code match the contract files exactly. Verified by: diff between contract definitions and implemented code shows zero deviations in names/types.
- **SC-002**: Self-review is eliminated. The Coder produces code and tests, then hands off to the Reviewer without self-assessment. Verified by: no self-review section in the agent's output.
- **SC-003**: Each implementation skill runs in a fresh context window, enabling deeper focus per phase. Verified by: each subagent loads only its SKILL.md, the WP file, and relevant contract files.
- **SC-004**: All tests pass before handoff to the Reviewer. Verified by: test runner exit code 0 with coverage at or above thresholds (80% code, 90% branch).
- **SC-005**: Adding a new implementation phase requires creating exactly one skill file in `.github/skills/code-*/SKILL.md`. Verified by: no coordinator edit needed.

---

## 3. Users & Roles

- **Human Developer (observer)**: May observe implementation progress but does not intervene unless escalated. The Coder operates autonomously after plan approval.

- **Planner Agent (upstream producer)**: Produces WP files and contract files that the Coder implements against. Each WP contains tasks with acceptance criteria and contract references.

- **Review Coordinator (downstream consumer)**: Reviews the Coder's output for spec adherence, quality, security, and test adequacy. The Coder hands off to the Reviewer after all tasks in a WP are complete.

- **Orchestrator Agent (invoker)**: Triggers the Coder when a plan is approved for implementation. Reads WP status to determine pipeline state.

---

## 4. Functional Requirements

### 4.1 Coder Coordinator

#### 4.1.1 Work Package Selection

- **FR-001**: The coordinator SHALL list all `.sdd/plans/WP*.md` files and select the specified WP (from argument) or present the list to the user via `vscode_askQuestions`.
  - Error: If no WP files exist, halt and inform the user.

- **FR-002**: Before starting, the coordinator SHALL read:
  1. The selected WP file in full
  2. `.sdd/plans/README.md` for sequencing context and dependency status
  3. The spec section(s) referenced in the WP
  4. Contract files in `.sdd/plans/contracts/<WP-slug>/`
  5. `AGENTS.md` at workspace root (if exists) for project-wide rules
  - Error: If a dependency WP has `lane` not equal to `done`, halt and recommend completing the dependency first.

- **FR-003**: The coordinator SHALL verify all contract files referenced by the WP's tasks exist and contain valid syntax. If any contract file is missing, halt and recommend re-running the Planner.

#### 4.1.2 Patterns Consumption

- **FR-004**: Before dispatching any skill, the coordinator SHALL read `.sdd/reviews/code-patterns.md` (if it exists) and include active pattern summaries in the prompt for each skill. Skills SHALL avoid producing code that would trigger known patterns.

#### 4.1.3 Dynamic Skill Discovery

- **FR-005**: The coordinator SHALL discover available coding skills by scanning for directories matching the glob pattern `.github/skills/code-*/SKILL.md` at the start of each WP implementation.
  - Postcondition: A sorted list of discovered skill names is produced.
  - Error: If zero skills are discovered, halt and report no coding skills installed.

- **FR-006**: The coordinator SHALL dispatch discovered skills in a deterministic order. The canonical order is:
  1. `code-env-setup` (environment setup, dependency installation, baseline verification)
  2. `code-implementation` (core implementation of all tasks in the WP)
  3. `code-unit-tests` (unit test writing for implemented code)
  4. `code-integration-tests` (integration test writing, component boundary testing)
  5. `code-debug` (conditional: debugging and fixing if tests fail)
  - Skills not present are skipped. Skills present but not in this list are dispatched after all known skills, in alphabetical order.

#### 4.1.4 Skill Dispatch

- **FR-007**: The coordinator SHALL dispatch each skill as a subagent invocation using `runSubagent`. Each invocation SHALL include:
  1. The skill file path
  2. The WP file path
  3. The contract files directory path for this WP
  4. The spec file path
  5. Active code-domain patterns to avoid
  6. The target language and framework
  7. Task list with acceptance criteria and spec refs
  - Error: If a skill fails (environment setup or implementation), halt the WP and report the error. The coordinator SHALL NOT proceed to later skills if a prerequisite fails.

- **FR-008**: Skills SHALL execute sequentially, one at a time, blocking. The coordinator SHALL NOT dispatch the next skill until the current skill completes.

- **FR-009**: Each skill SHALL read the current state of the codebase (files created or modified by prior skills) before executing. This enables context forwarding: the test skill sees code written by the implementation skill.

#### 4.1.5 Conditional Debug Skill

- **FR-010**: After the unit test and integration test skills complete, the coordinator SHALL check test results. If any tests fail:
  1. Dispatch the `code-debug` skill with failing test output, relevant source files, contract files, and spec refs.
  2. The debug skill SHALL diagnose failures, fix the code, and re-run tests.
  3. If tests still fail after the debug skill, the coordinator SHALL retry the debug skill up to 2 more times (max 3 debug attempts total).
  4. If tests still fail after 3 debug attempts, escalate to the human with full error context.
  - If all tests pass after the test skills, the debug skill is NOT dispatched.

#### 4.1.6 Task State Tracking

- **FR-011**: The coordinator SHALL update the WP file's `lane:` frontmatter to `doing` when implementation begins.

- **FR-012**: The coordinator SHALL track each task's progress using `manage_todo_list`. Each task in the WP SHALL be marked in-progress when its implementation starts and completed when its acceptance criteria are met.

- **FR-013**: The coordinator SHALL update the WP file after each task completes:
  1. Check off acceptance criteria (`- [ ]` to `- [x]`) as each criterion is verified
  2. Append an Activity Log entry with task ID, status, and timestamp

#### 4.1.7 Post-Completion Handoff

- **FR-014**: After all tasks are complete and all tests pass, the coordinator SHALL:
  1. Run a final coverage report and verify thresholds (80% code, 90% branch)
  2. Set the WP's `lane:` frontmatter to `for_review`
  3. Commit all changes with appropriate commit messages per task
  4. Hand off to the Review Coordinator
  - Note: NO self-review step occurs. The Reviewer is the quality gate.

- **FR-015**: The coordinator SHALL NOT perform any self-assessment, self-review, or quality evaluation of the code. The Coder's job ends at "tests pass + coverage met".

#### 4.1.8 Commit Policy

- **FR-016**: Each task SHALL be committed individually after completion:
  ```
  git add <explicit file list>
  git commit -m "<type>(<scope>): <description> (WP<NN> T<NN>-XX)"
  ```
  - Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`
  - Scope: module or feature area touched
  - Task ID always included at the end
  - Files SHALL be listed explicitly in `git add`.

#### Implementation Contract -- Coder Coordinator

**Inputs**:
- WP file path (string): the work package to implement.
- Contract files directory (string): `.sdd/plans/contracts/<WP-slug>/`.
- Spec file path (string): the specification referenced by the WP.
- Code-domain patterns (string, optional): from `code-patterns.md`.

**Outputs**:
- Implementation source files in the project directory structure.
- Test files (unit + integration) in the project test directory structure.
- Updated WP file with checked acceptance criteria and activity log.
- Git commits per task.
- WP `lane` set to `for_review`.

**Error behaviors**:
- No WP files: halt, inform user.
- Dependency WP not done: halt, recommend completing dependency.
- Missing contract files: halt, recommend re-running Planner.
- Environment setup failure: halt, ask user for help.
- Implementation skill failure: halt WP, report error.
- Tests fail after 3 debug attempts: escalate to human.

---

### 4.2 Coding Skills (Common Contract)

#### 4.2.1 Skill Input Contract

- **FR-017**: Every coding skill SHALL accept the following inputs in its subagent prompt:
  1. `skill_path`: Path to its SKILL.md file
  2. `wp_path`: Path to the WP file
  3. `contracts_dir`: Path to contract files for this WP
  4. `spec_path`: Path to the source spec
  5. `patterns`: Active code-domain patterns to avoid
  6. `target_language`: Programming language
  7. `target_framework`: Framework (e.g., Express, FastAPI, React)
  8. `task_list`: Tasks with acceptance criteria and spec refs

- **FR-018**: Every coding skill SHALL, upon invocation:
  1. Read its own SKILL.md
  2. Read the WP file and contract files
  3. Read the spec sections referenced by its tasks
  4. Execute its implementation work (code, tests, or debugging)
  5. Report results back to the coordinator

#### 4.2.2 Skill Output Contract

- **FR-019**: Each skill SHALL report to the coordinator:
  1. Files created or modified (list of paths)
  2. Tasks completed (list of task IDs)
  3. Test results (pass/fail counts, coverage percentage)
  4. Issues encountered (list of problems that need attention)
  5. Status: "success" or "failure" with reason

---

### 4.3 Environment Setup Skill (code-env-setup) -- Phase 1

- **FR-020**: The code-env-setup skill SHALL:
  1. Check for an existing virtual environment (venv, .venv, pyproject.toml, package.json, etc.)
  2. Create a virtual environment if none exists (Python: `python -m venv .venv`; Node: verify `node_modules/` from `npm install` or `yarn install`)
  3. Install project dependencies from the dependency manifest (requirements.txt, pyproject.toml, package.json)
  4. Run existing tests to verify the baseline is green
  5. Verify the application launches without errors (if applicable)
  6. Document environment state in the WP Activity Log

- **FR-021**: If the environment cannot be established (missing tools, dependency conflicts, service requirements), the skill SHALL:
  1. Document what failed and why
  2. Report failure to the coordinator
  3. The coordinator SHALL escalate to the human

- **FR-022**: The skill SHALL install the coverage tooling specified in the WP's foundation task (pytest-cov, istanbul/nyc, etc.) and configure minimum thresholds (80% code, 90% branch).

#### Implementation Contract -- code-env-setup

**Inputs**: WP file, target language/framework.
**Outputs**: Working development environment, dependency installation, baseline test verification.
**Error behaviors**: Environment creation failure - report to coordinator for human escalation.

---

### 4.4 Core Implementation Skill (code-implementation) -- Phase 2

- **FR-023**: The code-implementation skill SHALL implement all tasks in the WP in dependency order. For each task:
  1. Read the task's spec refs and acceptance criteria
  2. Read the contract files referenced by the task (interfaces, data schemas, API contracts, etc.)
  3. Implement code that satisfies every acceptance criterion
  4. Copy interface/type definitions from contract files verbatim into the implementation
  5. Implement all error paths and validation rules from the spec
  6. Follow existing codebase conventions (naming, structure, style)
  7. Check off acceptance criteria in the WP file as each is met

- **FR-024**: The skill SHALL implement contract-first:
  1. Function signatures SHALL match the contract's `interfaces.<ext>` exactly (parameter names, types, return types)
  2. Data entity fields SHALL match the contract's `data-schemas.<ext>` exactly (field names, types, defaults, validation)
  3. API endpoint paths, methods, request/response types SHALL match `api-contracts.<ext>` exactly
  4. State transitions SHALL match `state-machines.<ext>` exactly (valid states, guards, transitions)
  5. Error codes and messages SHALL match `error-catalog.<ext>` exactly

- **FR-025**: The skill SHALL NOT:
  1. Add features, abstraction layers, or configuration not specified in the spec or contracts
  2. Refactor unrelated code outside the task's scope
  3. Change contract definitions -- contracts are read-only
  4. Implement self-review or quality assessment

- **FR-026**: One `code-implementation` skill invocation SHALL handle all tasks in one WP. The skill processes tasks sequentially in dependency order within the single invocation.

#### Implementation Contract -- code-implementation

**Inputs**: WP file (tasks), contract files (interfaces, schemas, APIs, states, errors), spec sections.
**Outputs**: Implementation source files, updated WP file with checked acceptance criteria.
**Error behaviors**: Ambiguous requirement - flag via `[IMPL ISSUE: description]` in WP file, continue with best interpretation. Missing contract - halt, report to coordinator.

---

### 4.5 Unit Test Skill (code-unit-tests) -- Phase 3

- **FR-027**: The code-unit-tests skill SHALL write unit tests for all code implemented by the core implementation skill. Tests SHALL:
  1. Derive from spec acceptance scenarios (BDD approach), not from implementation details
  2. Cover happy path + error paths + edge cases per task
  3. Test real behavior through real code paths (no vacuous assertions)
  4. Mock only external dependencies, never the subject under test
  5. Use the project's test framework (pytest, Jest, etc.)

- **FR-028**: Every test SHALL be capable of failing. The skill SHALL NOT produce:
  1. `assert True` or equivalent trivial assertions
  2. Empty test bodies or `pass` stubs
  3. Tests that merely confirm a mock's return value

- **FR-029**: The skill SHALL run all unit tests after writing them and report results (pass/fail counts, coverage percentage).

- **FR-030**: The skill SHALL verify coverage meets thresholds:
  1. Code coverage at least 80%
  2. Branch coverage at least 90%
  - If coverage is below thresholds, the skill SHALL add more tests targeting uncovered lines/branches.

#### Implementation Contract -- code-unit-tests

**Inputs**: Implementation source files (from prior skill), WP file (acceptance scenarios), spec (BDD scenarios).
**Outputs**: Unit test files, test results report, coverage report.
**Error behaviors**: Tests fail - report failures to coordinator for debug skill dispatch. Coverage below thresholds - add more tests, re-verify.

---

### 4.6 Integration Test Skill (code-integration-tests) -- Phase 4

- **FR-031**: The code-integration-tests skill SHALL write integration tests for component boundaries:
  1. Tests across module boundaries within the WP's scope
  2. Tests against real database/storage if the WP sets up data persistence
  3. Tests against external API mocks (using the contract definitions to define mock responses)
  4. Data setup and teardown for each test

- **FR-032**: For external dependencies, the skill SHALL:
  1. Use contract files to generate mock responses that match exact schemas
  2. Test timeout, retry, and error handling paths
  3. Verify integration points match the API contract schemas

- **FR-033**: The skill SHALL run all integration tests after writing them and report results.

#### Implementation Contract -- code-integration-tests

**Inputs**: Implementation source files, unit test files, contract files (for mock generation), spec (integration test requirements).
**Outputs**: Integration test files, test results report.
**Error behaviors**: Tests fail - report to coordinator for debug skill dispatch.

---

### 4.7 Debug Skill (code-debug) -- Conditional Phase 5

- **FR-034**: The code-debug skill SHALL:
  1. Read failing test output (test names, error messages, stack traces)
  2. Read the corresponding source code
  3. Read the relevant contract files and spec sections
  4. Diagnose the root cause of each failure
  5. Fix the source code (or test code if the test is wrong)
  6. Re-run all tests to verify fixes
  7. Report results

- **FR-035**: The skill SHALL prioritize source code fixes over test code fixes. A test should only be modified if it genuinely tests the wrong behavior (not matching the spec). If the test correctly reflects the spec but the code is wrong, the code SHALL be fixed.

- **FR-036**: The skill SHALL NOT:
  1. Delete or skip failing tests
  2. Weaken assertions to make tests pass
  3. Add broad exception handlers to suppress errors
  4. Modify contract files

- **FR-037**: After fixes, the skill SHALL re-run all tests (unit + integration) and report:
  1. Previously failing tests that now pass
  2. Tests still failing (with diagnosis)
  3. New failures introduced by fixes (regression)

#### Implementation Contract -- code-debug

**Inputs**: Failing test output, source files, test files, contract files, spec sections.
**Outputs**: Fixed source/test files, re-run test results.
**Error behaviors**: Cannot diagnose failure - report to coordinator with full context for human escalation.

---

## 5. User Stories

### US-01 -- Implement a WP Contract-First (Priority: P1) MVP

**As the** Coder agent, **I want** to implement each task by reading contract files as the source of truth, **so that** my implementation matches the spec exactly without interpretation.

**Why P1**: Contract-first implementation is the core mechanism to eliminate drift.

**Independent Test**: Provide a WP with contract files (interfaces, schemas). Run the Coder. Verify: implemented function signatures match contract interfaces exactly, field names match data schemas, error codes match error catalog.

**Acceptance Scenarios**:
1. **Given** a WP with `interfaces.ts` defining `createUser(input: CreateUserInput): Promise<User>`, **When** the Coder implements, **Then** the implemented function has exactly that signature.
2. **Given** a WP with `data-schemas.ts` defining `User { id: string; email: string; role: 'admin' | 'user' }`, **When** the Coder implements, **Then** the User entity/model has exactly those fields with those types.
3. **Given** a WP with `error-catalog.ts` defining `USER_NOT_FOUND = { code: 'USR-001', status: 404 }`, **When** the Coder implements the user lookup, **Then** a 404 response with code 'USR-001' is returned when the user is not found.

---

### US-02 -- No Self-Review (Priority: P1) MVP

**As a** pipeline participant, **I want** the Coder to skip self-review entirely, **so that** the Reviewer provides a single, authoritative quality assessment.

**Why P1**: Self-review creates false security and duplicates the Reviewer's work.

**Independent Test**: Run the Coder on a WP. Verify: no self-review section in the output, no quality assessment, the WP goes directly to `for_review` status.

**Acceptance Scenarios**:
1. **Given** a completed WP implementation, **When** all tests pass, **Then** the Coder sets `lane: for_review` and hands off to the Reviewer without any review.
2. **Given** the Coder's skill sequence, **When** examining the canonical skill order, **Then** there is no review, assessment, or quality skill in the list.

---

### US-03 -- Automated Debugging on Test Failure (Priority: P1) MVP

**As the** Coder agent, **I want** a dedicated debug skill that diagnoses and fixes test failures, **so that** minor implementation errors are resolved without human intervention.

**Why P1**: Most test failures are minor bugs fixable by the agent itself.

**Independent Test**: Introduce a deliberate bug in implementation. Run the Coder. Verify: the debug skill catches and fixes the bug, tests pass after fix.

**Acceptance Scenarios**:
1. **Given** a unit test fails with a type mismatch, **When** the debug skill runs, **Then** it diagnoses the mismatch and fixes the source code, and re-run passes.
2. **Given** tests fail after 3 debug attempts, **When** the coordinator checks, **Then** it escalates to the human with full error context.
3. **Given** all tests pass after unit and integration test skills, **When** the coordinator checks, **Then** the debug skill is NOT dispatched.

---

### US-04 -- Skill-Based Implementation Pipeline (Priority: P1) MVP

**As the** Coder coordinator, **I want** each implementation phase as a separate skill with fresh context, **so that** each phase gets focused attention without context overload.

**Why P1**: Context overload in monolithic implementation causes missed details.

**Independent Test**: Run the Coder on a WP. Verify: 4-5 skill invocations occur (env setup, impl, unit tests, integration tests, optionally debug), each as a separate subagent call.

**Acceptance Scenarios**:
1. **Given** a WP with 8 tasks, **When** the Coder runs, **Then** code-env-setup runs first, code-implementation second, code-unit-tests third, code-integration-tests fourth.
2. **Given** code-env-setup fails (missing dependency), **When** the coordinator detects failure, **Then** it halts and does not dispatch code-implementation.

---

### US-05 -- Dynamic Coding Skill Discovery (Priority: P2)

**As a** system maintainer, **I want** to add a new coding phase by creating a `.github/skills/code-<name>/SKILL.md` file, **so that** the Coder discovers and dispatches it automatically.

**Why P2**: Extensibility for future phases without coordinator changes.

**Independent Test**: Add `.github/skills/code-linting/SKILL.md`. Run the Coder. Verify: the new skill is discovered and dispatched after known skills.

**Acceptance Scenarios**:
1. **Given** 5 coding skills, **When** a 6th `code-linting` is added, **Then** 6 skills are dispatched.

---

### Edge Cases

- What happens when a WP has no integration test requirements? The code-integration-tests skill produces no tests and reports success.
- What happens when contract files are empty (no entities, no endpoints for this WP)? The implementation skill follows task acceptance criteria from the WP prose, noting that no contracts applied.
- What happens when the debug skill introduces a regression? The debug re-runs all tests, catches the regression, and fixes it within its iteration budget.

---

## 6. User Flows

### 6.1 Full WP Implementation Flow

1. User or Orchestrator invokes the Coder with a WP ID.
2. Coordinator reads WP file, README, spec, contract files.
3. Coordinator verifies dependency WPs are done.
4. Coordinator verifies contract files exist and are valid.
5. Coordinator reads code-patterns.md (if exists).
6. Coordinator discovers coding skills via glob scan.
7. Dispatch code-env-setup: environment verification/creation.
8. Dispatch code-implementation: all tasks implemented contract-first.
9. Dispatch code-unit-tests: unit tests written and run.
10. Dispatch code-integration-tests: integration tests written and run.
11. Coordinator checks test results:
    a. All pass: proceed to step 12.
    b. Some fail: dispatch code-debug (up to 3 times).
    c. Still failing after 3 debugs: escalate to human, halt.
12. Coordinator runs final coverage report.
13. Coordinator sets WP `lane: for_review`.
14. Coordinator commits all changes.
15. Coordinator hands off to Review Coordinator.

### 6.2 Debug Flow

1. Tests fail after code-unit-tests or code-integration-tests.
2. Coordinator dispatches code-debug with failing test output.
3. Debug skill reads source code, contracts, spec.
4. Debug skill diagnoses root cause.
5. Debug skill fixes code.
6. Debug skill re-runs all tests.
7. Results reported to coordinator.
8. If still failing: coordinator increments attempt counter, dispatches code-debug again (max 3).
9. If 3 attempts exhausted: coordinator escalates to human.

---

## 7. Data Model

### 7.1 WP File State Transitions

The WP file's `lane` field tracks lifecycle:

| State | Meaning | Set By |
|-------|---------|--------|
| `planned` | Not yet started | Planner |
| `doing` | Implementation in progress | Coder coordinator (FR-011) |
| `for_review` | Implementation complete, awaiting review | Coder coordinator (FR-014) |
| `done` | Review passed | Reviewer |
| `to_do` | Review failed, needs rework | Reviewer |

Valid transitions:
- `planned` -> `doing` (Coder starts)
- `doing` -> `for_review` (Coder completes, tests pass)
- `for_review` -> `done` (Reviewer passes)
- `for_review` -> `to_do` (Reviewer fails)
- `to_do` -> `doing` (Coder reworks)

### 7.2 Task State Tracking

Within the WP file, tasks are tracked via acceptance criteria checkboxes:

| Symbol | Meaning |
|--------|---------|
| `- [ ]` | Not yet verified |
| `- [x]` | Verified and passing |

### 7.3 Activity Log Entry

Appended to the WP file after each task state change:

| Field | Type | Description |
|-------|------|-------------|
| Timestamp | datetime | When the state change occurred |
| Task ID | string | T<NN>-XX identifier |
| Status | enum | started, completed, blocked |
| Notes | string | Brief description of what was done |

### 7.4 Skill Result

Returned by each skill subagent to the coordinator:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| status | enum | `success`, `failure` | Skill outcome |
| files_modified | list(string) | file paths | Files created or changed |
| tasks_completed | list(string) | T<NN>-XX format | Tasks finished |
| test_results | object | pass_count, fail_count, coverage_pct | Test run summary |
| issues | list(string) | free text | Problems encountered |
| failure_reason | string | nullable | Why the skill failed (if failure) |

---

## 8. API / Interface Design

### 8.1 Coordinator Invocation Interface

**Invocation methods**:
1. Direct: user selects "Coder" agent mode with WP argument
2. Handoff: Orchestrator delegates after plan approval
3. Handoff: Planner hands off after plan is approved

**Response**: Implementation progress updates in VS Code chat, file modifications, commits, handoff to Reviewer.

### 8.2 Skill Subagent Prompt Template

```
Implement: <skill_name>

1. Read the skill instructions at: <skill_path>
2. Read the WP file at: <wp_path>
3. Read contract files at: <contracts_dir>
4. Read spec sections: <spec_refs>
5. Active patterns to avoid: <patterns>
6. Target: <target_language> with <target_framework>
7. Tasks: <task_list_with_acceptance_criteria>

Rules:
- Implement contract-first: signatures, types, fields MUST match contract files exactly
- Check off acceptance criteria in the WP file as you complete them
- Follow existing codebase conventions
- Do NOT add features not in the spec
- Do NOT perform self-review or quality assessment
- Report files modified, tasks completed, test results, and issues
```

### 8.3 Debug Skill Prompt Template

```
Debug failing tests.

1. Read the skill instructions at: <skill_path>
2. Failing test output:
<test_output>
3. Source files: <file_list>
4. Contract files at: <contracts_dir>
5. Spec refs: <spec_refs>

Diagnose root causes. Fix source code (prefer) or tests (only if test is wrong per spec).
Re-run ALL tests after fixes.
Do NOT delete tests, weaken assertions, or add broad exception handlers.
Report: fixed tests, still-failing tests, regressions.
Debug attempt: <N> of 3.
```

### 8.4 Handoff Prompt Templates

**Request Review** (to Reviewer):
```
WP<NN> implementation complete. All tests passing.
Coverage: <code_coverage>% code, <branch_coverage>% branch.
WP file: <wp_path>
Lane: for_review
```

**Clarify Specification** (to Spec Architect):
```
Spec ambiguity blocking implementation of task T<NN>-XX.
Issue: <description>
Spec ref: <FR-XXX>
```

---

## 9. Architecture

### 9.1 System Design

```
User/Orchestrator
       |
       v
Coder Coordinator
       |
       |--> Read WP + contracts + spec + patterns
       |--> Verify dependencies (prior WPs done)
       |--> Verify contract files exist
       |--> Discover skills (scan .github/skills/code-*/)
       |
       |--> runSubagent(code-env-setup)          --> env verified
       |--> runSubagent(code-implementation)      --> code written
       |--> runSubagent(code-unit-tests)          --> unit tests written + run
       |--> runSubagent(code-integration-tests)   --> integration tests written + run
       |
       |--> Check test results
       |     |--> All pass? --> coverage check --> for_review --> handoff to Reviewer
       |     |--> Fail? --> runSubagent(code-debug) (max 3x) --> re-check
       |                          |--> Still fail after 3? --> escalate to human
       |
       |--> Commit per task
       |--> Set lane: for_review
       |--> Handoff to Review Coordinator
```

### 9.2 Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Agent framework | VS Code Copilot Chat agents | Current | Existing SDD infrastructure |
| Skill framework | VS Code Copilot Chat skills | Current | Proven in Reviewer V2, Spec V2, Planner V2 |
| Contract files | Target language source | Varies | Read-only reference for implementation |
| Version control | Git | Current | Per-task commits with explicit file listing |

### 9.3 Directory & Module Structure

```
.github/
  agents/
    coder.agent.md                    # MODIFIED: refactored to coordinator pattern
  skills/
    code-env-setup/SKILL.md           # NEW: environment verification + setup
    code-implementation/SKILL.md      # NEW: contract-first task implementation
    code-unit-tests/SKILL.md          # NEW: unit test writing + execution
    code-integration-tests/SKILL.md   # NEW: integration test writing + execution
    code-debug/SKILL.md               # NEW: test failure diagnosis + fix

.sdd/
  plans/
    contracts/
      <WP-slug>/                      # READ-ONLY by Coder -- produced by Planner
        interfaces.<ext>
        data-schemas.<ext>
        api-contracts.<ext>
        state-machines.<ext>
        error-catalog.<ext>
  reviews/
    code-patterns.md                  # NEW: code-domain patterns
```

### 9.4 Key Design Decisions

**Decision 1: Remove self-review entirely**
- **Rationale**: User explicitly chose to trust the dedicated Reviewer agent. Self-review creates false security and wastes context window on duplicate analysis.
- **Alternatives considered**: Keep but simplify, replace with contract validation, keep with gate.
- **Consequences**: Reviewer becomes the sole quality gate. The Coder's job ends at "tests pass + coverage met".

**Decision 2: One core implementation skill per WP**
- **Rationale**: User chose this granularity. Decomposing implementation into per-task skills would create too many subagent invocations and lose cross-task context within a WP.
- **Alternatives considered**: One skill per task, hybrid (group related tasks).
- **Consequences**: The implementation skill must handle multi-task WPs (up to 12 tasks) within one invocation. Context forwarding within a single skill is natural (same context window).

**Decision 3: Separate test skills from implementation**
- **Rationale**: Testing in a fresh context window eliminates bias from implementation. The test skill reads implemented code as an external consumer.
- **Alternatives considered**: Tests inline with implementation, one test skill for all types.
- **Consequences**: Unit and integration tests are separate skills with separate context windows.

**Decision 4: Debug skill with 3-attempt budget**
- **Rationale**: Most test failures are minor bugs fixable by the agent. 3 attempts balances autonomy with escalation.
- **Alternatives considered**: No debug skill (always escalate), unlimited retries.
- **Consequences**: Reduces human interruptions for trivial bugs. Hard cap prevents infinite loops.

**Decision 5: Contract files are read-only**
- **Rationale**: Contracts are the Planner's output and the spec's derivative. The Coder implements against them, never changes them.
- **Alternatives considered**: Coder can propose contract amendments.
- **Consequences**: If a contract is wrong, the Coder must escalate to the Planner/Spec Architect.

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: A full WP implementation (5 skills) SHALL complete within 30 minutes for a medium-complexity WP (8 tasks, 800 lines of implementation code).
- **NFR-002**: Each debug attempt SHALL complete within 10 minutes.
- **NFR-003**: Test execution (unit + integration) SHALL complete within 5 minutes for medium-complexity WPs.

### 10.2 Security

- **NFR-004**: The Coder SHALL NOT commit secrets, tokens, credentials, or API keys to any file. Configuration SHALL use environment variables.
- **NFR-005**: Security-sensitive code (auth, encryption, input validation) SHALL follow OWASP guidelines and framework-specific security documentation.
- **NFR-006**: The Coder SHALL use web research to verify security patterns before implementing auth, encryption, or input validation.

### 10.3 Scalability & Availability

- Local workspace only. No availability or scaling requirements.
- **NFR-007**: The system SHALL handle WPs with up to 12 tasks and 2000 lines of implementation code.

### 10.4 Accessibility

- Not applicable at the agent level. Accessibility of implemented code is governed by the spec.

### 10.5 Observability

- **WP Activity Log**: Per-task state changes with timestamps.
- **Commit history**: Per-task commits for traceability.
- **Test results**: Coverage reports stored in project test output directory.

---

## 11. Test Requirements

### 11.1 Unit Tests

Validation SHALL verify:
- Coordinator agent file has valid YAML frontmatter + valid markdown.
- Each skill file has valid YAML frontmatter + valid markdown.
- Skill dispatch follows canonical order.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Coder V2 - Contract-First Implementation

  Scenario: Implement a WP using contract files
    Given a WP with 6 tasks and contract files (interfaces, schemas, errors)
    And 5 coding skills are installed
    When the Coder is invoked with the WP
    Then code-env-setup runs first
    And code-implementation implements all tasks
    And code-unit-tests writes and runs unit tests
    And code-integration-tests writes and runs integration tests
    And all tests pass
    And the WP lane is set to "for_review"
    And function signatures match contract interfaces exactly

  Scenario: No self-review
    Given a completed WP implementation
    When checking the Coder's output
    Then no self-review section exists
    And no quality assessment was performed
    And the WP went directly to for_review

  Scenario: Debug skill resolves test failures
    Given unit tests fail with a type mismatch
    When the Coder dispatches code-debug
    Then the debug skill diagnoses the mismatch
    And fixes the source code
    And re-runs all tests successfully

  Scenario: Escalation after 3 debug attempts
    Given tests fail due to a fundamental design issue
    When code-debug fails to fix after 3 attempts
    Then the Coder escalates to the human with full error context
    And does not mark the WP as for_review

  Scenario: Dependency check blocks premature implementation
    Given WP03 depends on WP02
    And WP02 lane is "doing" (not done)
    When the Coder is invoked with WP03
    Then it halts and recommends completing WP02 first

  Scenario: Missing contract files halt implementation
    Given a WP references interfaces.ts in its tasks
    But interfaces.ts does not exist in the contracts directory
    When the Coder starts
    Then it halts and recommends re-running the Planner

  Scenario: Environment setup failure
    Given the project requires Python 3.11 but only 3.8 is available
    When code-env-setup runs
    Then it reports the incompatibility
    And the coordinator escalates to the human

  Scenario: Coverage threshold enforcement
    Given unit tests pass but coverage is 72% (below 80%)
    When the coordinator checks coverage
    Then it dispatches code-unit-tests to add more tests
    And re-checks until coverage meets 80% code / 90% branch
```

### 11.3 Integration Tests

- Full pipeline: Planner V2 produces WP + contracts -> Coder V2 implements -> Reviewer reviews.
- Verify contract files are consumed correctly by the implementation skill.

### 11.4 End-to-End Tests

- Manual test: provide a real WP with contracts and run the Coder V2 through completion.

### 11.5 Performance Tests

- Time the full 5-skill dispatch on a medium-complexity WP. Target: under 30 minutes.

### 11.6 Security Tests

- Verify no credentials in committed files.
- Verify implemented security patterns match OWASP guidelines.

---

## 12. Constraints & Assumptions

### Constraints

- Must operate within the VS Code Copilot Chat agent framework.
- Skills execute sequentially.
- Contract files are read-only; Coder cannot modify them.
- The debug skill has a hard cap of 3 attempts.

### Assumptions

1. Planner V2 (Spec 003) is implemented and produces valid contract files.
2. Contract files in `.sdd/plans/contracts/` contain valid target-language syntax.
3. The target language and framework are specified in the WP or spec.
4. Test frameworks are available or installable via the environment setup skill.
5. Coverage tooling (pytest-cov, istanbul, etc.) is available for the target language.

---

## 13. Out of Scope

- **Self-review**: Explicitly removed. The Reviewer is the sole quality gate.
- **Spec writing or plan creation**: The Coder implements, not designs.
- **Contract modification**: Contracts are read-only. Issues go to the Planner.
- **Documentation generation**: Docs Agent (Spec 007) handles documentation.
- **Deployment**: The Coder implements, not deploys.

---

## 14. Open Questions

None remaining. All questions resolved:
1. One core implementation skill per WP (resolved in brief).
2. Self-review removed entirely (resolved in brief, Decision 5).
3. Debug skill max 3 attempts (resolved in FR-010).

---

## 15. Glossary

- **Contract file**: Language-specific source file from `.sdd/plans/contracts/` containing type definitions, interfaces, schemas, or constants. Read-only for the Coder.
- **Contract-first implementation**: Writing code that matches contract file definitions verbatim -- function signatures, field names, types, error codes.
- **Debug skill**: Conditional skill dispatched only when tests fail. Diagnoses and fixes failures.
- **Lane**: WP lifecycle state (planned, doing, for_review, done, to_do) tracked in YAML frontmatter.
- **Self-review**: Quality assessment by the implementation agent. Removed in V2 -- the Reviewer fills this role.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | WP selection | US-04 | Scenario 1 | BDD | 11.2 |
| FR-002 | Read WP + contracts + spec | US-01 | Scenario 1 | BDD | 11.2 |
| FR-003 | Contract file validation | US-01 | BDD Scenario 6 | BDD | 11.2 |
| FR-005 | Dynamic skill discovery | US-05 | Scenario 1 | BDD | 11.2 |
| FR-006 | Deterministic skill ordering | US-04 | Scenario 1, BDD Scenario 1 | BDD | 11.2 |
| FR-007 | Skill dispatch via runSubagent | US-04 | Scenario 1 | BDD | 11.2 |
| FR-010 | Conditional debug skill | US-03 | Scenario 1, 2, 3, BDD Scenario 3, 4 | BDD | 11.2 |
| FR-011 | WP lane to doing | US-04 | Scenario 1 | BDD | 11.2 |
| FR-014 | Post-completion handoff (for_review) | US-02 | Scenario 1, BDD Scenario 2 | BDD | 11.2 |
| FR-015 | No self-review | US-02 | Scenario 1, 2, BDD Scenario 2 | BDD | 11.2 |
| FR-020 | Environment setup skill | US-04 | Scenario 1, BDD Scenario 7 | BDD | 11.2 |
| FR-023 | Core implementation skill | US-01 | Scenario 1, 2, 3 | BDD | 11.2 |
| FR-024 | Contract-first implementation | US-01 | Scenario 1, 2, 3, BDD Scenario 1 | BDD | 11.2 |
| FR-025 | No over-engineering | US-01 | Scenario 1 | BDD | 11.2 |
| FR-027 | Unit test skill (BDD approach) | US-04 | Scenario 1 | BDD | 11.2 |
| FR-030 | Coverage threshold enforcement | US-04 | BDD Scenario 8 | BDD | 11.2 |
| FR-031 | Integration test skill | US-04 | Scenario 1 | BDD | 11.2 |
| FR-034 | Debug skill diagnosis + fix | US-03 | Scenario 1, BDD Scenario 3 | BDD | 11.2 |
| FR-035 | Prioritize source fixes over test fixes | US-03 | Scenario 1 | BDD | 11.2 |
| FR-036 | Debug shall not weaken/delete tests | US-03 | Scenario 1, BDD Scenario 4 | BDD | 11.2 |

---

## 17. Technical References

### Architecture & Patterns
- Contract-First Development, https://en.wikipedia.org/wiki/Design_by_contract, consulted 2026-04-05
- BDD/TDD best practices, https://cucumber.io/docs/bdd/, consulted 2026-04-05

### Technology Stack
- VS Code Copilot Chat agents documentation, https://code.visualstudio.com/docs/copilot, consulted 2026-04-05

### Testing
- pytest-cov documentation, https://pytest-cov.readthedocs.io/, consulted 2026-04-05
- Istanbul/nyc documentation, https://istanbul.js.org/, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.0.1 | 2026-04-05 | Spec Architect | Self-review corrections: verified self-review removal is explicit throughout, confirmed all FRs use SHALL, validated traceability matrix |
