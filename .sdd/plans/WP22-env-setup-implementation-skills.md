---
lane: planned
---

# WP22 - Environment Setup & Core Implementation Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/004-coder-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP20, WP21 |
| Goal | Implement the code-env-setup and code-implementation SKILL.md files that handle environment verification/setup and contract-first task implementation |
| Status | Not Started |
| Independent Test | Invoke the Coder on a WP with contract files. Verify: code-env-setup creates/verifies the environment and installs dependencies; code-implementation reads contracts and implements all tasks with matching signatures, fields, and error codes |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP22-env-setup-implementation-skills.md` |

## Objective

Implement the first two coding skills in the Coder V2 pipeline: `code-env-setup` (Phase 1) and `code-implementation` (Phase 2). The environment setup skill verifies or creates the development environment, installs dependencies, configures coverage tooling, and verifies the baseline. The core implementation skill implements all WP tasks contract-first, matching function signatures, field names, types, and error codes from contract files verbatim. These skills establish the "producing" phase of the pipeline -- they create the code that testing skills will verify.

## Spec References

FR-020, FR-021, FR-022 (code-env-setup), FR-023, FR-024, FR-025, FR-026 (code-implementation), FR-017, FR-018, FR-019 (common skill contract), Section 4.3 (Environment Setup), Section 4.4 (Core Implementation), Section 8.2 (Skill Prompt Template)

## Tasks

### T22-01 - Create code-env-setup SKILL.md structure

- **Description**: Replace the stub code-env-setup SKILL.md with the full skill file. Write the YAML frontmatter, input contract table (per CODER-SKILL-CONTRACT.md), execution sequence, and output format sections.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.2
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] SKILL.md has valid YAML frontmatter with name `code-env-setup` and description matching FR-006
  - [ ] Input contract table lists all 8 inputs from FR-017: skill_path, wp_path, contracts_dir, spec_path, patterns, target_language, target_framework, task_list
  - [ ] Execution sequence follows FR-018: read SKILL.md, read WP + contracts, read spec sections, execute work, report results
  - [ ] Output format matches FR-019: status, files_modified, tasks_completed, test_results, issues, failure_reason
  - [ ] Common contract reference to `.github/skills/CODER-SKILL-CONTRACT.md` is included
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Mirror structure from `.github/skills/plan-decomposition/SKILL.md` (input table, execution sequence, step-by-step instructions)
  - Files to modify: `.github/skills/code-env-setup/SKILL.md`

### T22-02 - Write environment detection and creation logic

- **Description**: Write the code-env-setup skill instructions for detecting existing environments and creating new ones when needed.
- **Spec refs**: FR-020
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The skill SHALL check for an existing virtual environment (venv, .venv, pyproject.toml, package.json, etc.) (FR-020.1)
  - [ ] The skill SHALL create a virtual environment if none exists: Python via `python -m venv .venv`; Node via `npm install` or `yarn install` (FR-020.2)
  - [ ] The skill SHALL install project dependencies from the dependency manifest (requirements.txt, pyproject.toml, package.json) (FR-020.3)
  - [ ] Given the project requires Python 3.11 but only 3.8 is available, the skill reports the incompatibility (BDD Scenario 7)
- **Test requirements**: BDD
- **Depends on**: T22-01
- **Implementation Guidance**:
  - Pattern: Check for environment indicators in priority order: `.venv/`, `venv/`, `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`
  - Known pitfalls: Must handle both fresh projects (no environment) and existing projects (environment exists but may need updates)
  - Error handling: Missing runtime -> report incompatibility with version details. Dependency conflict -> report exact error message.

### T22-03 - Write dependency installation and coverage tooling setup

- **Description**: Write the code-env-setup skill instructions for installing project dependencies and configuring coverage tooling with minimum thresholds.
- **Spec refs**: FR-020.3, FR-022
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The skill SHALL install the coverage tooling specified in the WP's foundation task (pytest-cov, istanbul/nyc, etc.) (FR-022)
  - [ ] The skill SHALL configure minimum thresholds: 80% code coverage, 90% branch coverage (FR-022)
  - [ ] Coverage configuration SHALL be written to the project's test configuration file (pytest.ini, .nycrc, jest.config.js, etc.)
- **Test requirements**: none
- **Depends on**: T22-02
- **Implementation Guidance**:
  - Official docs: https://pytest-cov.readthedocs.io/ (Python), https://istanbul.js.org/ (Node)
  - Spec validation rules: Thresholds are exactly 80% code and 90% branch -- not "at least" or "approximately"
  - Known pitfalls: Coverage tooling varies by language. The skill must detect the target language and install the appropriate tool.

### T22-04 - Write baseline verification and failure handling

- **Description**: Write the code-env-setup skill instructions for running existing tests to verify a green baseline, verifying application launch, documenting environment state, and handling setup failures.
- **Spec refs**: FR-020.4, FR-020.5, FR-020.6, FR-021
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The skill SHALL run existing tests to verify the baseline is green (FR-020.4)
  - [ ] The skill SHALL verify the application launches without errors if applicable (FR-020.5)
  - [ ] The skill SHALL document environment state in the WP Activity Log (FR-020.6)
  - [ ] If the environment cannot be established (missing tools, dependency conflicts, service requirements), the skill SHALL document what failed and why, report failure to the coordinator, and the coordinator SHALL escalate to the human (FR-021)
- **Test requirements**: BDD
- **Depends on**: T22-03
- **Implementation Guidance**:
  - Pattern: Run test command (e.g., `pytest`, `npm test`) and check exit code. Exit 0 = green baseline. Non-zero = report.
  - Error handling: The 3-part failure protocol from FR-021: (1) document, (2) report, (3) coordinator escalates
  - Known pitfalls: Some projects have no existing tests -- this is not a failure; document "no existing tests" and continue.

### T22-05 - Create code-implementation SKILL.md structure

- **Description**: Replace the stub code-implementation SKILL.md with the full skill file. Write the YAML frontmatter, input contract table, execution sequence, output format, and constraint sections.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] SKILL.md has valid YAML frontmatter with name `code-implementation` and description matching FR-006
  - [ ] Input contract table lists all 8 inputs from FR-017
  - [ ] Execution sequence follows FR-018
  - [ ] Output format matches FR-019
  - [ ] Common contract reference to `.github/skills/CODER-SKILL-CONTRACT.md` is included
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Same structure as T22-01 but for the implementation skill
  - Files to modify: `.github/skills/code-implementation/SKILL.md`

### T22-06 - Write contract-first implementation logic

- **Description**: Write the code-implementation skill instructions for implementing all tasks in a WP contract-first. The skill SHALL copy interface/type definitions from contract files verbatim and implement code that satisfies every acceptance criterion.
- **Spec refs**: FR-023, FR-024
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The skill SHALL implement all tasks in dependency order; for each task: read spec refs, read contract files, implement code satisfying every acceptance criterion, copy interface/type definitions verbatim, implement all error paths, follow codebase conventions, check off acceptance criteria (FR-023)
  - [ ] Function signatures SHALL match the contract's `interfaces.<ext>` exactly (parameter names, types, return types) (FR-024.1)
  - [ ] Data entity fields SHALL match the contract's `data-schemas.<ext>` exactly (field names, types, defaults, validation) (FR-024.2)
  - [ ] API endpoint paths, methods, request/response types SHALL match `api-contracts.<ext>` exactly (FR-024.3)
  - [ ] State transitions SHALL match `state-machines.<ext>` exactly (valid states, guards, transitions) (FR-024.4)
  - [ ] Error codes and messages SHALL match `error-catalog.<ext>` exactly (FR-024.5)
  - [ ] Given a contract defining `createUser(input: CreateUserInput): Promise<User>`, the implemented function has exactly that signature (US-01 Scenario 1)
- **Test requirements**: BDD
- **Depends on**: T22-05
- **Implementation Guidance**:
  - Pattern: For each task, read its contract file references, then implement matching the contract definitions character-for-character
  - Known pitfalls: "Verbatim" means exact -- no renaming, no retyping, no adding optional parameters not in the contract
  - Error handling: Missing contract file -> halt and report to coordinator. Ambiguous requirement -> flag with `[IMPL ISSUE: description]` in WP file, continue with best interpretation.

### T22-07 - Write implementation constraints and scope rules

- **Description**: Write the code-implementation skill constraints: no over-engineering, no out-of-scope refactoring, contracts are read-only, no self-review. Write the single-invocation-per-WP rule.
- **Spec refs**: FR-025, FR-026
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The skill SHALL NOT add features, abstraction layers, or configuration not specified in the spec or contracts (FR-025.1)
  - [ ] The skill SHALL NOT refactor unrelated code outside the task's scope (FR-025.2)
  - [ ] The skill SHALL NOT change contract definitions -- contracts are read-only (FR-025.3)
  - [ ] The skill SHALL NOT implement self-review or quality assessment (FR-025.4)
  - [ ] One `code-implementation` skill invocation SHALL handle all tasks in one WP; tasks are processed sequentially in dependency order (FR-026)
- **Test requirements**: none
- **Depends on**: T22-06
- **Implementation Guidance**:
  - Pattern: Write these as explicit "SHALL NOT" constraint rules in the SKILL.md
  - Known pitfalls: These constraints must be prominent and unambiguous -- they prevent scope creep and contract violation
  - Spec validation rules: Decision 5 from Section 9.4: "Contracts are the Planner's output. The Coder implements against them, never changes them."

### T22-08 - Integration verification of both skills with coordinator

- **Description**: Verify that both code-env-setup and code-implementation SKILL.md files are correctly discovered by the coordinator's glob pattern, their input contracts match the coordinator's dispatch template, and their output contracts match the coordinator's result handling.
- **Spec refs**: FR-005, FR-007, FR-017, FR-019
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Both skills are discovered by `file_search` with glob `.github/skills/code-*/SKILL.md`
  - [ ] Input contract fields match the coordinator's dispatch template from Section 8.2
  - [ ] Output contract fields match the coordinator's expected result format from FR-019 and Section 7.4
  - [ ] No files contain em dashes, smart quotes, or curly apostrophes
- **Test requirements**: none
- **Depends on**: T22-07
- **Implementation Guidance**:
  - Pattern: Cross-reference coordinator dispatch template (WP21) with skill input tables
  - Use `grep -rP '[\x{2013}\x{2014}\x{2018}\x{2019}\x{201C}\x{201D}]'` to check for prohibited characters

## Implementation Notes

- Both skills are markdown SKILL.md files containing instructions for the AI subagent, not executable code.
- The code-env-setup skill must be language-agnostic in its instructions -- it should detect the target language and apply the appropriate setup steps.
- The code-implementation skill is the most critical skill in the pipeline: it must enforce contract-first implementation rigorously. Any deviation from contract definitions is a defect.
- Decision 2 from Section 9.4: One implementation skill per WP (not per task). The skill handles multi-task WPs within a single invocation.
- The implementation skill's error handling uses `[IMPL ISSUE: description]` markers for ambiguous requirements, allowing the coordinator to surface these to the user.

## Parallel Opportunities

- T22-01 (env-setup structure) and T22-05 (implementation structure) can run in parallel [P]
- All other tasks are sequential within their skill

## Risks & Mitigations

- **Risk**: Contract-first instructions may be too rigid for edge cases where contracts are incomplete. **Mitigation**: The `[IMPL ISSUE]` marker allows flagging ambiguities without blocking.
- **Risk**: Environment setup instructions may not cover all language ecosystems. **Mitigation**: Cover the most common (Python, Node, Go, Rust) and include a generic fallback pattern.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
