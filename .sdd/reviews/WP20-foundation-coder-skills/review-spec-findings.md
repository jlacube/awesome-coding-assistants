---
skill: review-spec
wp: WP20-foundation-coder-skills
spec: .sdd/specs/004-coder-v2.spec.md
reviewed_at: 2026-04-05T13:00:00Z
status: completed
finding_counts:
  pass: 7
  warn: 0
  fail: 0
  na: 11
files_reviewed:
  - .github/skills/code-env-setup/SKILL.md
  - .github/skills/code-implementation/SKILL.md
  - .github/skills/code-unit-tests/SKILL.md
  - .github/skills/code-integration-tests/SKILL.md
  - .github/skills/code-debug/SKILL.md
  - .github/skills/CODER-SKILL-CONTRACT.md
  - .sdd/reviews/code-patterns.md
  - .github/agents/coder.agent.md
  - .sdd/reviews/review-patterns.md
---

# review-spec Findings for WP20-foundation-coder-skills

## Summary

WP20 is a foundation/scaffolding WP that creates directory structure, stub skill files, a common skill contract document, and a code-patterns placeholder. It produces no functional behavior -- all artifacts are markdown files that enable WP21-WP24 to implement the Coder V2 skill-based pipeline.

**In-scope FRs**: FR-005 (dynamic skill discovery pattern), FR-006 (canonical skill names), FR-017 (8-input contract), FR-018 (5-step execution sequence), FR-019 (output contract), Section 8.2 (prompt template), Section 9.3 (directory structure).

**Out-of-scope FRs**: FR-001-FR-004, FR-007-FR-016, FR-020-FR-037 (implemented in WP21-WP24).

**Overall assessment**: 7 PASS, 0 FAIL, 11 N/A. All in-scope FRs are fully satisfied by the scaffolding artifacts. The 5 skill directories match the glob pattern and canonical names exactly. The CODER-SKILL-CONTRACT.md faithfully codifies FR-017, FR-018, FR-019, and Section 8.2. The directory structure matches Section 9.3. All N/A items are either runtime behaviors not applicable to a scaffolding WP, success criteria that require coordinator logic from later WPs, or FRs explicitly out of scope.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-005
- **File**: .github/skills/code-env-setup/SKILL.md, .github/skills/code-implementation/SKILL.md, .github/skills/code-unit-tests/SKILL.md, .github/skills/code-integration-tests/SKILL.md, .github/skills/code-debug/SKILL.md
- **Description**: FR-005 requires the coordinator to discover coding skills by scanning `.github/skills/code-*/SKILL.md`. All 5 directories exist with valid SKILL.md files containing proper YAML frontmatter (name, description, argument-hint). The glob pattern `.github/skills/code-*/SKILL.md` matches all 5 files. WP20's scope is creating the discoverable targets; the actual scanning logic is WP21.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-006
- **File**: .github/skills/CODER-SKILL-CONTRACT.md#L80-L90
- **Description**: FR-006 defines 5 canonical skill names and their dispatch order. All 5 directories use the exact canonical names: code-env-setup, code-implementation, code-unit-tests, code-integration-tests, code-debug. The CODER-SKILL-CONTRACT.md Section 5 documents the canonical dispatch order with matching phase numbers and purposes. The dispatching logic itself is WP21; WP20 establishes the naming convention and documents the order.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-017
- **File**: .github/skills/CODER-SKILL-CONTRACT.md#L10-L23
- **Description**: FR-017 requires every coding skill to accept 8 inputs. CODER-SKILL-CONTRACT.md Section 1 defines all 8 inputs in a table matching FR-017 exactly: skill_path (Path), wp_path (Path), contracts_dir (Path), spec_path (Path), patterns (Text), target_language (String), target_framework (String), task_list (Text). Names, types, and descriptions align with the spec.

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-018
- **File**: .github/skills/CODER-SKILL-CONTRACT.md#L27-L35
- **Description**: FR-018 requires every coding skill to execute 5 steps in order. CODER-SKILL-CONTRACT.md Section 2 defines the 5 steps matching FR-018 exactly: (1) Read SKILL.md, (2) Read WP + contracts, (3) Read spec sections, (4) Execute implementation work, (5) Report results. Step descriptions align with the spec text.

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-019
- **File**: .github/skills/CODER-SKILL-CONTRACT.md#L39-L51
- **Description**: FR-019 requires each skill to report 5 categories of information. CODER-SKILL-CONTRACT.md Section 3 defines 6 fields that cover all 5 categories: status (enum: success/failure), files_modified, tasks_completed, test_results (object with pass_count, fail_count, coverage_pct per Section 7.4), issues, and failure_reason (nullable). FR-019 item 5 ("Status: success or failure with reason") is correctly decomposed into the status enum + failure_reason string, matching the spec's own Section 7.4 data model which also has 6 fields.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation (prompt template)
- **Requirement**: Section 8.2
- **File**: .github/skills/CODER-SKILL-CONTRACT.md#L55-L74
- **Description**: The WP acceptance criteria for T20-03 require the prompt template from Section 8.2 to be included verbatim. The CODER-SKILL-CONTRACT.md Section 4 contains the template with identical structure, placeholders, numbered steps, and rules. All 7 numbered items and all 6 rules match the spec text exactly.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation (directory structure)
- **Requirement**: Section 9.3
- **File**: .github/skills/, .sdd/reviews/code-patterns.md, .github/agents/coder.agent.md
- **Description**: Section 9.3 specifies the directory and module structure for the Coder V2. All files listed as NEW or MODIFIED in Section 9.3 exist at their specified paths: 5 skill SKILL.md files (NEW), code-patterns.md (NEW), and coder.agent.md (MODIFIED with coordinator-pattern frontmatter). Additionally, CODER-SKILL-CONTRACT.md was created as an organizational artifact paralleling existing PLAN-SKILL-CONTRACT.md and SPEC-SKILL-CONTRACT.md patterns, serving as the developer guide for FR-017/FR-018/FR-019.

### SPEC-008 [N/A]
- **Checklist item**: Preconditions enforced, Postconditions produced, Error paths handled - FR-005, FR-006
- **Justification**: FR-005 postcondition ("sorted list of discovered skill names") and error path ("zero skills discovered -> halt") are coordinator runtime behaviors implemented in WP21. FR-006 dispatch logic is also WP21. WP20 only creates the static scaffolding that these runtime behaviors operate on.

### SPEC-009 [N/A]
- **Checklist item**: Preconditions enforced, Error paths handled, Edge cases - FR-017, FR-018, FR-019
- **Justification**: These FRs define the contract that coding skills must follow at runtime. WP20 documents the contract in CODER-SKILL-CONTRACT.md but does not implement runtime enforcement. Preconditions (e.g., validating inputs exist), error paths (e.g., missing contract files), and edge cases (e.g., empty contracts) are enforced when skills are implemented in WP22-WP24 and when the coordinator dispatches in WP21.

### SPEC-010 [N/A]
- **Checklist item**: Data model match - all in-scope FRs
- **Justification**: No runtime data model instantiation occurs in this scaffolding WP. The CODER-SKILL-CONTRACT.md defines the Skill Result data model fields matching Section 7.4 (status, files_modified, tasks_completed, test_results, issues, failure_reason), but no code produces or consumes these structures. Runtime data model validation is deferred to WP21-WP24.

### SPEC-011 [N/A]
- **Checklist item**: API contract match - all in-scope FRs
- **Justification**: No API endpoints exist in this WP. All artifacts are markdown files (skill stubs, contract documentation, patterns placeholder, agent frontmatter). The only "API" is the skill subagent prompt template documented in CODER-SKILL-CONTRACT.md Section 4, which is verified as matching Section 8.2 in SPEC-006.

### SPEC-012 [N/A]
- **Checklist item**: Error codes match - all in-scope FRs
- **Justification**: No error codes are produced or consumed in this scaffolding WP. Error handling behavior is documented in CODER-SKILL-CONTRACT.md Section 7 but not enforced at runtime until WP21-WP24.

### SPEC-013 [N/A]
- **Checklist item**: SC verification - SC-001
- **Justification**: SC-001 requires implementation code to align with contract files. WP20 creates no implementation code -- only scaffolding artifacts. SC-001 is verifiable after WP22 (code-implementation skill) implements against contract files.

### SPEC-014 [N/A]
- **Checklist item**: SC verification - SC-002
- **Justification**: SC-002 requires self-review to be eliminated. The coordinator logic that would (or would not) include self-review is implemented in WP21. WP20 only updates the agent frontmatter. Note: the coder.agent.md body still contains legacy self-review references from pre-V2, but the body content is explicitly preserved per T20-05 and will be replaced by WP21's coordinator logic.

### SPEC-015 [N/A]
- **Checklist item**: SC verification - SC-003
- **Justification**: SC-003 requires each skill to run in a fresh context window. Skills are stubs (intentionally, per T20-02). Fresh-context verification requires the coordinator to dispatch skills via runSubagent, which is WP21.

### SPEC-016 [N/A]
- **Checklist item**: SC verification - SC-004
- **Justification**: SC-004 requires all tests to pass before handoff with coverage thresholds. WP20 has no executable code or tests -- all artifacts are markdown files. This SC applies to WPs that produce implementation code (WP22+).

### SPEC-017 [N/A]
- **Checklist item**: SC verification - SC-005
- **Justification**: SC-005 requires adding a new implementation phase by creating exactly one skill file. WP20 establishes the directory convention and glob pattern (`.github/skills/code-*/SKILL.md`) that enables this, but full verification requires the coordinator's dynamic discovery logic in WP21 to confirm that a new `code-*/SKILL.md` is automatically discovered and dispatched.

### SPEC-018 [N/A]
- **Checklist item**: FR classification - out of scope
- **Justification**: FR-001-FR-004 (WP selection, input reading, contract verification, patterns consumption), FR-007-FR-016 (skill dispatch, sequential execution, context forwarding, conditional debug, task tracking, post-completion handoff, commit policy), and FR-020-FR-037 (individual skill implementations) are all out of scope for WP20. These FRs are implemented in WP21 (coordinator), WP22 (env-setup + implementation skills), WP23 (test skills), and WP24 (debug skill).
