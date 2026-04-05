---
lane: done
---

# WP20 - Foundation: Coder Skill Scaffolding

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/004-coder-v2.spec.md` |
| Priority | P0 |
| Lane | for_review |
| Depends on | none |
| Goal | Create the directory structure, stub skill files, common skill contract, and code-patterns placeholder so all Coder V2 skills can be implemented |
| Status | Complete |
| Independent Test | Verify: 5 coding skill directories exist under `.github/skills/code-*/`, each contains a stub `SKILL.md` with valid YAML frontmatter, `CODER-SKILL-CONTRACT.md` defines the 8-input contract, and `code-patterns.md` exists |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP20-foundation-coder-skills.md` |

## Objective

Establish the foundation directories, stub files, common skill contract, and patterns placeholder required by all 5 coding skills and the Coder coordinator. This WP creates no functional behavior -- it builds the scaffolding that WP21-WP24 implement against. Mirrors the pattern established in WP08 (Reviewer V2 foundation) and WP14 (Planner V2 foundation).

## Spec References

FR-005, FR-017, FR-018, FR-019, Section 9.3 (Directory Structure), Section 9.2 (Technology Stack), Section 8.2 (Skill Subagent Prompt Template)

## Tasks

### T20-01 - Create coding skill directories

- **Description**: Create 5 directories under `.github/skills/` matching the canonical skill names from FR-006: `code-env-setup`, `code-implementation`, `code-unit-tests`, `code-integration-tests`, `code-debug`.
- **Spec refs**: FR-005, FR-006, Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] All 5 directories exist under `.github/skills/`: `code-env-setup/`, `code-implementation/`, `code-unit-tests/`, `code-integration-tests/`, `code-debug/`
  - [x] Directory names match the canonical names from FR-006 exactly
  - [x] No extra directories created beyond the 5 specified
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Official docs: https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview
  - Pattern: Follow the same structure as existing `code-*` or `spec-*` skill directories
  - Known pitfalls: Directory names must match the glob pattern `.github/skills/code-*/SKILL.md` used in FR-005 for dynamic discovery
  - Spec validation rules: Names are lowercase-kebab-case with `code-` prefix

### T20-02 - Create stub SKILL.md files with YAML frontmatter

- **Description**: Create a stub `SKILL.md` in each of the 5 coding skill directories. Each stub SHALL have valid YAML frontmatter with `name`, `description`, and `argument-hint` fields. The body SHALL contain a placeholder comment indicating the skill is not yet implemented.
- **Spec refs**: FR-005, Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each of the 5 directories contains a `SKILL.md` file
  - [x] Each `SKILL.md` has valid YAML frontmatter with `name`, `description`, and `argument-hint`
  - [x] `argument-hint` reads "Invoked by Coder Coordinator - do not call directly"
  - [x] The `name` field matches the directory name (e.g., `code-env-setup`)
  - [x] The `description` field matches the canonical purpose from FR-006
- **Test requirements**: none
- **Depends on**: T20-01
- **Implementation Guidance**:
  - Pattern: Mirror the frontmatter format from `.github/skills/plan-decomposition/SKILL.md`:
    ```yaml
    ---
    name: code-env-setup
    description: "Environment verification, dependency installation, baseline test verification"
    argument-hint: "Invoked by Coder Coordinator - do not call directly"
    ---
    ```
  - Known pitfalls: YAML frontmatter must start on line 1 with `---` and end with `---`. No leading whitespace.

### T20-03 - Create CODER-SKILL-CONTRACT.md

- **Description**: Create `.github/skills/CODER-SKILL-CONTRACT.md` defining the common input/output contract for all `code-*` skills. This contract parallels `PLAN-SKILL-CONTRACT.md` and `SPEC-SKILL-CONTRACT.md` but with the 8-input spec from FR-017 and the output contract from FR-019.
- **Spec refs**: FR-017, FR-018, FR-019, Section 8.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] File exists at `.github/skills/CODER-SKILL-CONTRACT.md`
  - [x] Contract specifies all 8 inputs from FR-017: skill_path, wp_path, contracts_dir, spec_path, patterns, target_language, target_framework, task_list
  - [x] Contract specifies the 5-step execution sequence from FR-018: read SKILL.md, read WP + contracts, read spec sections, execute implementation work, report results
  - [x] Contract specifies the output fields from FR-019: status, files_modified, tasks_completed, test_results, issues, failure_reason
  - [x] Contract includes the skill subagent prompt template from Section 8.2 verbatim
- **Test requirements**: none
- **Depends on**: T20-02
- **Implementation Guidance**:
  - Pattern: Mirror `.github/skills/PLAN-SKILL-CONTRACT.md` structure
  - Files to create: `.github/skills/CODER-SKILL-CONTRACT.md`
  - Spec validation rules: Output status is enum `success` or `failure`. test_results object has pass_count, fail_count, coverage_pct fields per Section 7.4.

### T20-04 - Create code-patterns.md placeholder

- **Description**: Create `.sdd/reviews/code-patterns.md` as a placeholder patterns file for the Coder domain. The coordinator reads this file before dispatching skills (FR-004). Initialize with "Active Patterns" and "Resolved" sections, both empty.
- **Spec refs**: FR-004, Section 9.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] File exists at `.sdd/reviews/code-patterns.md`
  - [x] File has a header and "Active Patterns" and "Resolved" sections
  - [x] "Active Patterns" section is initially empty with "(none)" placeholder
  - [x] File follows the same format as `.sdd/reviews/review-patterns.md`
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Copy the structure from `.sdd/reviews/review-patterns.md`
  - Files to create: `.sdd/reviews/code-patterns.md`

### T20-05 - Refactor coder.agent.md YAML frontmatter

- **Description**: Update `.github/agents/coder.agent.md` YAML frontmatter to adopt the coordinator pattern. Set the agent name, description, tools (runSubagent, vscode_askQuestions, manage_todo_list, file system, terminal), and handoff buttons (Request Review -> Reviewer, Clarify Specification -> Spec Architect). Do NOT write the coordinator logic body yet -- that is WP21's responsibility.
- **Spec refs**: FR-007, Section 8.1, Section 8.4, Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] `coder.agent.md` has valid YAML frontmatter starting on line 1
  - [x] `tools` list includes `runSubagent` for skill dispatch (FR-007)
  - [x] `tools` list includes `vscode_askQuestions` for WP selection (FR-001)
  - [x] `tools` list includes `manage_todo_list` for task tracking (FR-012)
  - [x] Handoff buttons defined: "Request Review" (Reviewer) and "Clarify Specification" (Spec Architect) per Section 8.4
  - [x] Existing coordinator body content is preserved (not deleted)
- **Test requirements**: none
- **Depends on**: T20-03
- **Implementation Guidance**:
  - Pattern: Mirror `.github/agents/spec-architect.agent.md` and `.github/agents/planner.agent.md` frontmatter structure
  - Official docs: https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview
  - Known pitfalls: Agent name must match the expected handoff target from Orchestrator and Planner agents. Check existing handoff references before renaming.
  - Files to modify: `.github/agents/coder.agent.md`

### T20-06 - Verify directory structure and encoding compliance

- **Description**: Verify all files created in T20-01 through T20-05 exist, have correct paths, contain valid YAML, and use plain ASCII encoding (no em dashes, smart quotes, or curly apostrophes). Commit all foundation files.
- **Spec refs**: Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] All 5 skill directories exist and each contains a SKILL.md
  - [x] CODER-SKILL-CONTRACT.md exists at `.github/skills/CODER-SKILL-CONTRACT.md`
  - [x] code-patterns.md exists at `.sdd/reviews/code-patterns.md`
  - [x] coder.agent.md has updated YAML frontmatter
  - [x] No files contain em dashes, smart quotes, or curly apostrophes
- **Test requirements**: none
- **Depends on**: T20-05
- **Implementation Guidance**:
  - Use `grep -rP '[\x{2013}\x{2014}\x{2018}\x{2019}\x{201C}\x{201D}]'` to check for prohibited characters
  - Commit message: `chore(coder): scaffold coding skill directories and contracts (WP20 T20-06)`

## Implementation Notes

- All artifacts are markdown files. No executable code, build system, or test framework is involved.
- The foundation WP establishes the glob pattern `.github/skills/code-*/SKILL.md` that the coordinator relies on for dynamic discovery (FR-005).
- The CODER-SKILL-CONTRACT.md defines the interface between the coordinator and all skills. Each skill WP (WP22-WP24) implements this contract.
- The code-patterns.md file is consumed by FR-004 and populated by the Reviewer after code reviews.

## Parallel Opportunities

- T20-04 (code-patterns.md) can run in parallel with T20-01 through T20-03
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: Existing coder.agent.md has content that must not be lost during frontmatter refactoring. **Mitigation**: Read full file before editing; preserve body content.
- **Risk**: Skill directory naming mismatch with glob pattern. **Mitigation**: Use exact names from FR-006; verify with `ls .github/skills/code-*/SKILL.md`.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T12:00:00Z - coder - lane=doing - Starting implementation, markdown-only WP, no env setup needed
- 2026-04-05T12:15:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T23:00:00Z - review-coordinator - lane=done - Verdict: Approved

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T23:00:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked
- [PASS] Activity Log: Consistent lane transitions
- [PASS] Commit granularity: 5 commits for 6 tasks (T20-01+T20-02 combined appropriately)
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. No action required.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 7 | 0 | 0 |
| **Total** | **11** | **0** | **0** |
