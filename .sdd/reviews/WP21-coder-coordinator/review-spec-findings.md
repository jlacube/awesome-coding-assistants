---
skill: review-spec
wp: WP21-coder-coordinator
spec: .sdd/specs/004-coder-v2.spec.md
reviewed_at: 2026-04-05T17:00:00Z
status: completed
finding_counts:
  pass: 27
  warn: 0
  fail: 2
  na: 3
files_reviewed:
  - .github/agents/coder.agent.md
  - .sdd/specs/004-coder-v2.spec.md
  - .sdd/plans/WP21-coder-coordinator.md
---

# review-spec Findings for WP21-coder-coordinator

## Summary

Evaluated 16 functional requirements (FR-001 through FR-016), 9 spec sections (6.1, 6.2, 7.1, 8.1-8.4, 9.1, 9.4), and 5 success criteria (SC-001 through SC-005) against the coordinator body in `.github/agents/coder.agent.md`.

Overall assessment: **Strong compliance**. All 16 FRs are fully implemented with correct obligations, preconditions, postconditions, and error paths. Two template deviations found in Sections 8.2 and 8.3 where the implementation adds extra rules not present in the spec-defined templates. Three items classified N/A (SC-001 evaluates skill output not the coordinator; data model and error code checklist items do not apply to a markdown coordinator file).

**FR Classification**: 16 Compliant, 0 Partial, 0 Deviating, 0 Missing.
**Section Classification**: 7 Compliant, 2 Deviating (8.2, 8.3).
**SC Classification**: 4 Compliant, 1 N/A.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-001
- **File**: .github/agents/coder.agent.md#L67-L73
- **Description**: WP selection is fully implemented. Step 1 uses `list_dir` to scan `.sdd/plans/` for `WP*.md` files, loads directly if a WP ID argument is provided, presents the list via `vscode_askQuestions` if no WP is specified, and halts with "No WP files found in .sdd/plans/. Run the Planner first." if the directory is empty. All obligations, the error path, and the user interaction scenario are satisfied.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation, preconditions, error paths
- **Requirement**: FR-002
- **File**: .github/agents/coder.agent.md#L75-L83
- **Description**: The 5-item artifact chain is fully implemented in Step 2. The coordinator reads: (1) the selected WP file, (2) `.sdd/plans/README.md` for sequencing context, (3) spec sections referenced in the WP's `Spec` field, (4) contract files in `.sdd/plans/contracts/<WP-slug>/`, and (5) `AGENTS.md` at workspace root (with graceful handling if missing). The dependency check reads each dependency WP's YAML frontmatter `lane:` field and halts if not `done`, with the exact error message format specified.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation, error paths
- **Requirement**: FR-003
- **File**: .github/agents/coder.agent.md#L85-L91
- **Description**: Contract file validation is fully implemented in Step 3. The coordinator parses task descriptions for contract file references, uses `list_dir` on the contracts directory, verifies each referenced file exists, reads each file to verify valid syntax, and halts with an appropriate error if any are missing. The edge case of an empty/missing contracts directory with no contract references is handled (proceed without error).

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-004
- **File**: .github/agents/coder.agent.md#L93-L98
- **Description**: Patterns consumption is fully implemented in Step 4. The coordinator reads `.sdd/reviews/code-patterns.md`, extracts the "Active Patterns" section if the file exists, sets patterns to "No active patterns" if the file does not exist (no error), and stores the active patterns text for inclusion in every skill dispatch prompt (Step 6 template includes `<patterns>` substitution).

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation, postconditions, error paths
- **Requirement**: FR-005
- **File**: .github/agents/coder.agent.md#L100-L117
- **Description**: Dynamic skill discovery is fully implemented in Step 5. The coordinator uses `file_search` with glob `.github/skills/code-*/SKILL.md`, extracts skill names from directory paths, produces a sorted list, and halts with "No coding skills are installed" if zero skills are discovered.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-006
- **File**: .github/agents/coder.agent.md#L107-L117
- **Description**: Canonical dispatch order is correctly implemented. The table in Step 5 lists the exact canonical order: (1) code-env-setup, (2) code-implementation, (3) code-unit-tests, (4) code-integration-tests, (5) code-debug. Skills from the canonical list that are not present are skipped without error. Skills present but not in the canonical list are dispatched after all known skills in alphabetical order.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation, error paths
- **Requirement**: FR-007 (coordinator sends FR-017 inputs)
- **File**: .github/agents/coder.agent.md#L125-L157
- **Description**: Skill dispatch via `runSubagent` is fully implemented in Step 6. The prompt template includes all 8 inputs from FR-017: (1) skill_path, (2) wp_path, (3) contracts_dir, (4) spec_path, (5) patterns, (6) target_language, (7) target_framework (items 6-7 combined on one line but both substitution values documented separately), (8) task_list_with_acceptance_criteria. The substitution values section explicitly lists all 8 values. Failure handling halts the WP and does not dispatch remaining skills.

### SPEC-008 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-008
- **File**: .github/agents/coder.agent.md#L119-L122
- **Description**: Sequential blocking dispatch is explicitly stated: "Dispatch each discovered skill... one at a time using `runSubagent`. Skills execute sequentially, blocking. Do NOT dispatch the next skill until the current skill completes."

### SPEC-009 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-009
- **File**: .github/agents/coder.agent.md#L155-L157
- **Description**: Context forwarding is addressed: "Each skill reads the current state of the codebase (files created or modified by prior skills) before executing. This is automatic since each subagent reads the filesystem fresh." This correctly implements FR-009's requirement that each skill reads current codebase state.

### SPEC-010 [PASS]
- **Checklist item**: FR classification - SHALL obligation, error paths
- **Requirement**: FR-010
- **File**: .github/agents/coder.agent.md#L160-L197
- **Description**: Conditional debug with retry logic is fully implemented in Step 7. The coordinator checks test results after unit and integration test skills. If all pass, debug is skipped. If any fail, `code-debug` is dispatched with the debug prompt template. The attempt counter starts at 1, retries up to 3 total. After 3 failures, escalation to human includes: failing test names/messages, relevant source files, contract files, spec references, and summary of all 3 debug attempt outcomes. The WP is NOT marked `for_review` on escalation.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-011
- **File**: .github/agents/coder.agent.md#L201-L204
- **Description**: Lane update to `doing` on implementation start is implemented in Step 8a. The coordinator sets `lane:` YAML frontmatter to `doing` and appends an Activity Log entry with timestamp. This is explicitly gated after all validation (Steps 1-5) passes.

### SPEC-012 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-012
- **File**: .github/agents/coder.agent.md#L206-L209
- **Description**: Task tracking via `manage_todo_list` is implemented in Step 8b. Tasks are marked in-progress when started and completed when ACs are met. The rules section also includes "ALWAYS use #tool:todo to track every task in the work package -- mark each in-progress and completed as you go" (line 56).

### SPEC-013 [PASS]
- **Checklist item**: FR classification - SHALL obligation, postconditions
- **Requirement**: FR-013
- **File**: .github/agents/coder.agent.md#L211-L218
- **Description**: WP file updates after task completion are implemented in Step 8c. Acceptance criteria checkboxes are checked off (`- [ ]` to `- [x]`). Activity Log entries include task ID, status, and timestamp in the format `<timestamp> - coder - <task_id> - completed - <brief notes>`, matching Section 7.3's entry format.

### SPEC-014 [PASS]
- **Checklist item**: FR classification - SHALL obligation, postconditions
- **Requirement**: FR-014
- **File**: .github/agents/coder.agent.md#L236-L256
- **Description**: Post-completion handoff is fully implemented in Step 9. The coordinator runs a final coverage report verifying 80% code / 90% branch thresholds, re-dispatches test skills if below thresholds, sets `lane:` to `for_review`, appends an Activity Log entry, updates the plan index in README.md, and hands off to the Reviewer using the handoff template from Section 8.4. The commit step is handled in Step 10.

### SPEC-015 [PASS]
- **Checklist item**: FR classification - SHALL NOT obligation
- **Requirement**: FR-015
- **File**: .github/agents/coder.agent.md#L248-L250
- **Description**: The FR-015 SHALL NOT obligation is comprehensively enforced. Step 9 includes an explicit paragraph: "The coordinator SHALL NOT perform any self-assessment, self-review, or quality evaluation of the code. No review checklists, no quality scores, no 'verified implementation quality' statements." The rules section (line 33) also includes "NEVER perform self-review, self-assessment, or quality evaluation of code." No self-review language appears anywhere in the coordinator body.

### SPEC-016 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-016
- **File**: .github/agents/coder.agent.md#L258-L298
- **Description**: Per-task commit policy is fully implemented in Step 10 and the `<commit_policy>` section. The format `<type>(<scope>): <description> (WP<NN> T<NN>-XX)` is specified. All 6 commit types (feat, fix, refactor, test, docs, chore) are listed. Files must be listed explicitly in `git add` -- never `git add .` or `git add -A`. Task ID is always included at the end. The rules section also enforces "NEVER use `git add .` or `git add -A` -- always list files explicitly" (line 37).

### SPEC-017 [PASS]
- **Checklist item**: Section 6.1 - Full WP Implementation Flow
- **Requirement**: Section 6.1
- **File**: .github/agents/coder.agent.md#L67-L298
- **Description**: All 15 steps of the spec's Section 6.1 flow are implemented across Steps 1-10 of the coordinator. The sequence is: WP selection (Step 1), artifact chain + dependency check (Step 2), contract validation (Step 3), patterns (Step 4), skill discovery (Step 5), sequential dispatch (Step 6), test check + debug (Step 7), state tracking (Step 8), coverage + for_review (Step 9), commit + handoff (Step 10). The flow order matches the spec exactly.

### SPEC-018 [PASS]
- **Checklist item**: Section 6.2 - Debug Flow
- **Requirement**: Section 6.2
- **File**: .github/agents/coder.agent.md#L160-L197
- **Description**: All 9 steps of the spec's Section 6.2 debug flow are implemented in Step 7. Tests fail -> dispatch code-debug with output -> debug reads source/contracts/spec -> diagnoses root cause -> fixes code -> re-runs all tests -> reports to coordinator -> retry if failing (max 3) -> escalate after 3.

### SPEC-019 [PASS]
- **Checklist item**: Section 7.1 - WP File State Transitions
- **Requirement**: Section 7.1
- **File**: .github/agents/coder.agent.md#L201-L256
- **Description**: The Coder-owned state transitions are correctly implemented: `planned` -> `doing` (Step 8a, line 203), `doing` -> `for_review` (Step 9, line 243). The Activity Log protocol (lines 225-233) specifies valid lanes matching the spec's state diagram. Transitions owned by the Reviewer (`for_review` -> `done`/`to_do`) are not the Coder's responsibility and are correctly excluded from the Coder's set operations.

### SPEC-020 [PASS]
- **Checklist item**: Section 8.1 - Coordinator Invocation Interface
- **Requirement**: Section 8.1
- **File**: .github/agents/coder.agent.md#L1-L73
- **Description**: The coordinator supports all three invocation methods from Section 8.1: (1) Direct invocation via agent mode with WP argument (YAML frontmatter `argument-hint` and Step 1 line 69), (2) Orchestrator handoff (Step 1 accepts any invocation), (3) Planner handoff (Step 1 accepts any invocation). Response is implementation progress in chat, file modifications, commits, and handoff to Reviewer.

### SPEC-021 [FAIL]
- **Checklist item**: API contract match - Section 8.2 Skill Subagent Prompt Template
- **Requirement**: Section 8.2
- **File**: .github/agents/coder.agent.md#L136-L143
- **Description**: The skill prompt template in Step 6 deviates from the verbatim spec template. One extra rule line is inserted that does not appear in the spec's Section 8.2 template.
- **Expected**: The Rules section of the prompt template should match Section 8.2 exactly:
  ```
  Rules:
  - Implement contract-first: signatures, types, fields MUST match contract files exactly
  - Check off acceptance criteria in the WP file as you complete them
  - Follow existing codebase conventions
  - Do NOT add features not in the spec
  - Do NOT perform self-review or quality assessment
  - Report files modified, tasks completed, test results, and issues
  ```
- **Evidence**:
  ```
  Rules:
  - Implement contract-first: signatures, types, fields MUST match contract files exactly
  - Contract files are READ-ONLY -- do NOT modify any file in .sdd/plans/contracts/
  - Check off acceptance criteria in the WP file as you complete them
  - Follow existing codebase conventions
  - Do NOT add features not in the spec
  - Do NOT perform self-review or quality assessment
  - Report files modified, tasks completed, test results, and issues
  ```
  Line 138 (`Contract files are READ-ONLY...`) is an addition not present in the spec template. While this reinforces spec Decision 5 (contracts read-only) and FR-003, it alters the API contract between coordinator and skills beyond what the spec defines in Section 8.2.

### SPEC-022 [FAIL]
- **Checklist item**: API contract match - Section 8.3 Debug Skill Prompt Template
- **Requirement**: Section 8.3
- **File**: .github/agents/coder.agent.md#L180-L187
- **Description**: The debug prompt template in Step 7 deviates from the verbatim spec template in two places: (1) a clarifying parenthetical is added, and (2) an extra rule line is inserted.
- **Expected**: The spec's Section 8.3 template has:
  ```
  Re-run ALL tests after fixes.
  Do NOT delete tests, weaken assertions, or add broad exception handlers.
  Report: fixed tests, still-failing tests, regressions.
  ```
- **Evidence**:
  ```
  Re-run ALL tests (unit + integration) after fixes.
  Do NOT delete tests, weaken assertions, or add broad exception handlers.
  Do NOT modify contract files -- they are read-only.
  Report: fixed tests, still-failing tests, regressions.
  ```
  Line 182 adds `(unit + integration)` clarification not in the spec template. Line 184 adds `Do NOT modify contract files -- they are read-only.` which is not in the spec template. Both additions are spec-aligned (FR-037 says "re-run all tests (unit + integration)" and Decision 5 mandates contracts read-only), but they alter the API contract beyond what Section 8.3 defines.

### SPEC-023 [PASS]
- **Checklist item**: API contract match - Section 8.4 Handoff Prompt Templates
- **Requirement**: Section 8.4
- **File**: .github/agents/coder.agent.md#L252-L256, #L316-L326
- **Description**: Both handoff templates from Section 8.4 are included verbatim. "Request Review" template (Step 9, lines 252-256) matches the spec exactly: `WP<NN> implementation complete. All tests passing. Coverage: <code_coverage>% code, <branch_coverage>% branch. WP file: <wp_path> Lane: for_review`. "Clarify Specification" template (Step 11, lines 322-326) matches the spec exactly: `Spec ambiguity blocking implementation of task T<NN>-XX. Issue: <description> Spec ref: <FR-XXX>`.

### SPEC-024 [PASS]
- **Checklist item**: Section 9.1 - System Design architecture
- **Requirement**: Section 9.1
- **File**: .github/agents/coder.agent.md#L67-L298
- **Description**: The implementation follows the architecture from Section 9.1 exactly. The flow matches: read WP + contracts + spec + patterns -> verify dependencies -> verify contracts -> discover skills -> sequential runSubagent dispatch (env-setup, implementation, unit-tests, integration-tests) -> check test results (all pass -> coverage -> for_review -> handoff; fail -> debug max 3x -> escalate) -> commit per task -> set lane for_review -> handoff to Reviewer.

### SPEC-025 [PASS]
- **Checklist item**: Section 9.4 - Key Design Decisions
- **Requirement**: Section 9.4 (Decisions 1-5)
- **File**: .github/agents/coder.agent.md#L29-L59
- **Description**: All 5 design decisions are reflected in the implementation. Decision 1 (no self-review): enforced by rules line 33 and Step 9 FR-015 section. Decision 2 (one skill per WP): Step 6 dispatches one invocation per skill phase, implementation skill handles all tasks. Decision 3 (separate test skills): unit and integration tests are separate skills in canonical order. Decision 4 (debug 3 attempts): Step 7 implements 3-attempt budget with escalation. Decision 5 (contracts read-only): rules line 32 states "NEVER modify contract files in `.sdd/plans/contracts/`".

### SPEC-026 [N/A]
- **Checklist item**: SC-001 - Implementation aligns with contract files
- **Justification**: SC-001 evaluates the output of coding skills (whether implemented function signatures, field names, types match contract files). The coordinator dispatches skills but does not produce implementation code itself. Verification requires runtime observation of skill output, which is outside the scope of a coordinator body review.

### SPEC-027 [PASS]
- **Checklist item**: SC-002 - Self-review eliminated
- **Requirement**: SC-002
- **File**: .github/agents/coder.agent.md#L33, #L248-L250
- **Description**: Self-review is comprehensively eliminated. The rules section includes "NEVER perform self-review, self-assessment, or quality evaluation of code" (line 33). Step 9 includes an explicit FR-015 block prohibiting all self-assessment language. The canonical skill order contains no review or assessment skill. No self-review section, quality score, or assessment language appears anywhere in the coordinator body.

### SPEC-028 [PASS]
- **Checklist item**: SC-003 - Fresh context per skill
- **Requirement**: SC-003
- **File**: .github/agents/coder.agent.md#L119-L157
- **Description**: Each skill is dispatched as a separate `runSubagent` invocation (Step 6), which creates a fresh context window. The prompt template instructs each skill to read its own SKILL.md, the WP file, and contract files. Context forwarding is documented as automatic via filesystem reads (line 155-157).

### SPEC-029 [PASS]
- **Checklist item**: SC-004 - All tests pass before handoff
- **Requirement**: SC-004
- **File**: .github/agents/coder.agent.md#L160-L242
- **Description**: The coordinator checks test results after test skills (Step 7) and runs a final coverage report (Step 9). Coverage thresholds (80% code, 90% branch) are verified. If below thresholds, test skills are re-dispatched. The WP is only set to `for_review` after all tests pass and coverage is met.

### SPEC-030 [PASS]
- **Checklist item**: SC-005 - Extensibility via single skill file
- **Requirement**: SC-005
- **File**: .github/agents/coder.agent.md#L100-L117
- **Description**: Dynamic skill discovery in Step 5 uses `file_search` with glob `.github/skills/code-*/SKILL.md`. Any new skill matching this glob pattern is automatically discovered and dispatched after canonical skills in alphabetical order. No coordinator edit is needed to add a new implementation phase.

### SPEC-031 [N/A]
- **Checklist item**: Data model match (Section 7)
- **Justification**: The coordinator is a markdown instruction file, not an executable code module. The data model in Section 7 defines WP file state fields, task tracking symbols, activity log entry format, and skill result structures. These are consumed as markdown conventions within the coordinator's instructions (e.g., `- [ ]` checkboxes, activity log format). There are no data entity types, database schemas, or programmatic data structures to validate against Section 7. The coordinator's usage of these conventions is verified as part of FR-011/FR-012/FR-013 findings above.

### SPEC-032 [N/A]
- **Checklist item**: Error codes match
- **Justification**: The spec does not define an error code taxonomy or error catalog for the coordinator. All error behaviors are descriptive text messages (e.g., "No WP files found", "Dependency WP<NN> has lane=<value>", "Contract file <path> is missing"). These error messages are verified as part of individual FR error path checks (SPEC-001 through SPEC-010).
