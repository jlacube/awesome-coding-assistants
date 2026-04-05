---
skill: review-spec
wp: WP15-planner-coordinator
spec: .sdd/specs/003-planner-v2.spec.md
reviewed_at: 2026-04-05T12:00:00Z
status: completed
finding_counts:
  pass: 29
  warn: 0
  fail: 3
  na: 5
files_reviewed:
  - .github/agents/planner.agent.md
  - .sdd/specs/003-planner-v2.spec.md
  - .sdd/plans/WP15-planner-coordinator.md
---

# review-spec Findings for WP15-planner-coordinator

## Summary

Evaluated 22 functional requirements (FR-001 through FR-022) from spec Section 4.1 and 6 success criteria (SC-001 through SC-006) against the implementation in `.github/agents/planner.agent.md`. Of 22 FRs: 19 are Compliant, 2 are Deviating, 1 is Partial. 4 success criteria verified as PASS, 2 deferred (N/A). 3 cross-cutting checklist items are N/A (data model, API contract, error codes do not apply to an agent instruction file).

Overall: the coordinator is well-structured and faithfully implements the vast majority of the spec. Three deviations were found: (1) FR-008 web research item 4 substitutes CI/CD patterns for starter templates, (2) FR-014 Phase 2 prompt template omits two inputs required by the FR's SHALL statement (though it matches the spec's own Section 8.3 template verbatim), (3) FR-019 ambiguous language check list omits the key term "should" while adding unlisted terms.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-001
- **File**: .github/agents/planner.agent.md#L62-L70
- **Description**: The coordinator lists all files in `.sdd/specs/` via `list_dir`, presents via `vscode_askQuestions` for multiple specs, confirms for single spec, and halts with an informative message if the directory is empty. All three branches (empty, single, multiple) are handled as specified.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - Error path
- **Requirement**: FR-001
- **File**: .github/agents/planner.agent.md#L65
- **Description**: Empty `.sdd/specs/` directory is handled: "No specs found in .sdd/specs/. Create a spec first using the Spec Architect agent." and halts. Matches the spec's error behavior.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-002
- **File**: .github/agents/planner.agent.md#L68-L69
- **Description**: The coordinator reads the selected spec in full (item 5) and reads companion artifacts from `.sdd/specs/artifacts/<NNN>-<idea-name>/` if the directory exists (item 6). Both actions match the spec.

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-003
- **File**: .github/agents/planner.agent.md#L70
- **Description**: Status validation verifies "Validated" or "Final". If "Draft", the coordinator refuses to proceed and recommends handing off to the Spec Architect. Matches the spec exactly.

### SPEC-005 [PASS]
- **Checklist item**: FR classification - Error path
- **Requirement**: FR-003
- **File**: .github/agents/planner.agent.md#L70
- **Description**: Draft spec error path is handled: refuses to proceed and recommends Spec Architect handoff. Matches spec error behavior.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-004
- **File**: .github/agents/planner.agent.md#L72-L100
- **Description**: All 7 completeness checks are present: (1) traceability matrix, (2) error behaviors, (3) data validation rules, (4) API error codes, (5) integration failure strategies, (6) state machines, (7) cross-cutting concerns. If any check fails, a structured gap report is created with the specified columns (Gap ID, Category, FR Reference, Description, Impact).

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-005
- **File**: .github/agents/planner.agent.md#L92-L96
- **Description**: Companion artifact consistency is verified against the prose spec with three sub-checks: field names in data model artifacts match Section 7, endpoint signatures in API artifacts match Section 8, and error codes match Section 4. Matches the spec.

### SPEC-008 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-006
- **File**: .github/agents/planner.agent.md#L103-L131
- **Description**: Auto-loop invokes Spec Architect via `runSubagent` with gap report, spec path, and artifacts directory. Retries up to 3 times. After 3 failures, escalates to human. On subagent failure, escalates with full context. After success, re-reads spec and re-runs completeness pre-check. Prompt template matches Section 8.4.

### SPEC-009 [PASS]
- **Checklist item**: FR classification - Error path
- **Requirement**: FR-006
- **File**: .github/agents/planner.agent.md#L126-L129
- **Description**: Three error paths are handled: (1) gaps remain after attempt -> retry, (2) 3 failed attempts -> escalate to human, (3) subagent failure -> escalate to human with full context. All match spec error behaviors.

### SPEC-010 [PASS]
- **Checklist item**: FR classification - Postcondition
- **Requirement**: FR-006
- **File**: .github/agents/planner.agent.md#L124-L125
- **Description**: Postcondition satisfied: after successful auto-loop, coordinator re-reads spec (step 2) and re-runs completeness pre-check (step 3) as specified.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-007
- **File**: .github/agents/planner.agent.md#L135-L149
- **Description**: Workspace research subagent is dispatched via `runSubagent` with agent "Explore". Discovers existing code, project structure, build system, test frameworks, patterns, conventions, and existing plans. Matches all 3 discovery targets from the spec.

### SPEC-012 [PASS]
- **Checklist item**: FR classification - Constraint
- **Requirement**: FR-007
- **File**: .github/agents/planner.agent.md#L148
- **Description**: The research subagent prompt explicitly states "Do NOT draft any plan content -- discovery and feasibility only." This matches the spec's constraint that the research subagent SHALL NOT draft plan content.

### SPEC-013 [FAIL]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-008
- **File**: .github/agents/planner.agent.md#L155-L159
- **Description**: Web research item 4 deviates from the spec. The implementation substitutes a different research target than what FR-008 specifies.
- **Expected**: FR-008 item 4: "Starter templates or boilerplate repos matching the tech stack"
- **Evidence**:
  ```markdown
  Conduct web research using `fetch_webpage` for:
  1. Official docs for libraries and frameworks in the spec's tech stack
  2. Known pitfalls, gotchas, and migration issues
  3. Testing framework guides and recommended patterns
  4. CI/CD best practices for the target platform
  ```
  Item 4 should be "Starter templates or boilerplate repos matching the tech stack" per FR-008, but the implementation says "CI/CD best practices for the target platform". The same deviation appears in the `web_research_policy` section (line 47).

### SPEC-014 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-009
- **File**: .github/agents/planner.agent.md#L163-L168
- **Description**: The coordinator reads `.sdd/reviews/plan-patterns.md` if it exists and includes active patterns in skill prompts. If the file does not exist, it proceeds without error. Matches the spec.

### SPEC-015 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-010
- **File**: .github/agents/planner.agent.md#L181-L184
- **Description**: Dynamic skill discovery uses `file_search` with glob pattern `.github/skills/plan-*/SKILL.md`. Skill names are extracted from directory paths. Produces a sorted list. Matches the spec.

### SPEC-016 [PASS]
- **Checklist item**: FR classification - Error path
- **Requirement**: FR-010
- **File**: .github/agents/planner.agent.md#L186
- **Description**: Zero skills discovered -> halts with informative message: "No plan skills are installed. Install at least one plan skill in .github/skills/plan-*/SKILL.md." Matches spec error behavior.

### SPEC-017 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-011
- **File**: .github/agents/planner.agent.md#L189-L203
- **Description**: All 8 skills listed in canonical order matching the spec exactly: plan-decomposition, plan-acceptance (Phase 1), then plan-interface-contracts, plan-data-schemas, plan-api-contracts, plan-state-machines, plan-error-catalogs, plan-cross-wp-validation (Phase 2). Missing skills are skipped without error. Unknown skills are dispatched after canonical skills in alphabetical order.

### SPEC-018 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-012
- **File**: .github/agents/planner.agent.md#L205-L261
- **Description**: Two-phase execution is implemented: Phase 1 (Steps 8) dispatches decomposition skills first, producing WP files and README. Phase 2 (Step 9) reads the plan accumulator and generates contract files. Output paths match spec: `.sdd/plans/` for Phase 1, `.sdd/plans/contracts/<WP-slug>/` for Phase 2.

### SPEC-019 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-013
- **File**: .github/agents/planner.agent.md#L253
- **Description**: The Phase 2 template rule states "Stay within 800 lines per contract file; split if needed." The WP implementation guidance (T15-08) notes "The 800-line block limit (FR-013) is enforced by Phase 2 skills themselves, not by the coordinator." The coordinator communicates the constraint to skills as required.

### SPEC-020 [FAIL]
- **Checklist item**: FR classification - SHALL obligation (Partial)
- **Requirement**: FR-014
- **File**: .github/agents/planner.agent.md#L241-L257
- **Description**: FR-014 states "Each invocation SHALL include" 7 items. The Phase 2 prompt template (Step 9) omits two of these required inputs: (4) research findings summary and (6) active plan-domain patterns to avoid. The Phase 1 template (Step 8) includes all items from FR-014 except contracts_dir (which is listed in FR-023 but not FR-014).
- **Expected**: FR-014 requires each invocation to include: (1) skill file path, (2) plan accumulator paths, (3) spec file path and companion artifacts, (4) research findings summary, (5) target language, (6) active plan-domain patterns, (7) phase indicator. The Phase 2 template omits items 4 and 6.
- **Evidence**:
  ```markdown
  Execute Phase 2 contract generation: <skill_name>

  1. Read the skill instructions at: <skill_path>
  2. Read the spec at: <spec_path> and artifacts at: <spec_artifacts_dir>
  3. Read the plan at: <plan_dir> (README + WP files)
  4. Target language: <target_language>
  5. Contracts directory: <contracts_dir>
  ```
  Missing: `research_summary` and `patterns` inputs. Note: the implementation matches the spec's own Section 8.3 template verbatim. The conflict is between FR-014's SHALL statement and Section 8.3's template definition -- a spec-internal inconsistency. The implementation faithfully follows Section 8.3.

### SPEC-021 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-015
- **File**: .github/agents/planner.agent.md#L232-L233
- **Description**: Phase 1 skills execute sequentially and block. Phase 1 failure halts immediately. Phase 2 skills execute sequentially but skip failed skills. Matches the spec.

### SPEC-022 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-016
- **File**: .github/agents/planner.agent.md#L218-L248
- **Description**: Both Phase 1 template (item 4: "Read existing plan state at: <plan_dir>") and Phase 2 template (item 3: "Read the plan at: <plan_dir>") instruct skills to read current plan state before writing. Phase 1 rules also state "Read existing plan files to maintain consistency with prior skills."

### SPEC-023 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-017
- **File**: .github/agents/planner.agent.md#L170-L180
- **Description**: Plan accumulator initialization creates: (1) `.sdd/plans/` if not exists, (2) `.sdd/plans/contracts/` if not exists, (3) skeleton README with spec reference, target language, and plan status "In Progress". All three spec requirements are met. Note: implementation also creates `.sdd/plans/contracts/shared/` which is not specified in FR-017 but is additive (supports shared entity pattern referenced in Phase 2 template rules).

### SPEC-024 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-018
- **File**: .github/agents/planner.agent.md#L267-L273
- **Description**: All 7 cross-WP consistency checks are present: (1) data contract consistency, (2) API/interface contract consistency, (3) dependency integrity with no circular deps, (4) configuration consistency, (5) test consistency at 80% code/90% branch, (6) spec traceability with every FR assigned to exactly one task, (7) contract-to-task alignment. Inconsistencies are fixed inline and documented in README under "Consistency Notes."

### SPEC-025 [FAIL]
- **Checklist item**: FR classification - SHALL obligation (Deviating)
- **Requirement**: FR-019
- **File**: .github/agents/planner.agent.md#L280
- **Description**: The ambiguous language check list deviates from the spec. The implementation omits the key term "should" (which is critical for distinguishing SHALL vs. SHOULD obligations in specs) and adds terms not listed in FR-019.
- **Expected**: FR-019 item 5: No ambiguous language ("should", "appropriate", "reasonable")
- **Evidence**:
  ```markdown
  5. No ambiguous language ("appropriate", "reasonable", "as needed", "etc.", "similar")
  ```
  Missing: "should" (specified in FR-019). Added but not in spec: "as needed", "etc.", "similar". The omission of "should" is significant because the should/shall distinction is a fundamental spec quality gate.

### SPEC-026 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-020
- **File**: .github/agents/planner.agent.md#L286-L295
- **Description**: After validation, the coordinator presents the plan to the user in chat. The presentation includes a summary table of WPs, MVP scope, dependency graph, task count totals, and any warnings. Matches the spec.

### SPEC-027 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-021
- **File**: .github/agents/planner.agent.md#L296-L301
- **Description**: All three feedback types are handled: (1) approval -> acknowledge and recommend Coder for WP01, (2) changes requested -> revise WPs and re-validate, (3) questions -> clarify or ask follow-ups. The handoff prompt to Coder matches Section 8.5 template (defined in YAML frontmatter handoffs section).

### SPEC-028 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-022
- **File**: .github/agents/planner.agent.md#L303-L323
- **Description**: Commit policy implemented correctly: (1) each WP file committed individually with explicit `git add`, (2) README committed as standalone change, (3) contract files committed per-WP. The rules section enforces "NEVER use `git add .` or `git add -A`" (line 37) and commit message format follows `docs(plan):` convention. Matches the spec.

### SPEC-029 [N/A]
- **Checklist item**: Data model match
- **Justification**: Not applicable. The implementation is an agent instruction file (markdown), not executable code. There are no data model implementations to verify against spec Section 7.

### SPEC-030 [N/A]
- **Checklist item**: API contract match
- **Justification**: Not applicable. The implementation is an agent instruction file (markdown) that orchestrates via `runSubagent` prompts. There are no API request/response schemas to verify against spec Section 8.

### SPEC-031 [N/A]
- **Checklist item**: Error codes match
- **Justification**: Not applicable. Error handling in the agent file is instructional text (e.g., "halt and report", "escalate to human"), not error code returns. No error taxonomy applies.

### SPEC-032 [N/A]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-001 (every task references contract artifacts)
- **Justification**: Deferred verification. SC-001 is a runtime outcome of plan generation, not verifiable from the static coordinator file. Requires executing the Planner against a spec and inspecting the generated WP files.

### SPEC-033 [N/A]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-002 (contract files contain valid syntax)
- **Justification**: Deferred verification. SC-002 depends on Phase 2 skill execution producing syntactically valid contract files. Not verifiable from the coordinator instructions alone.

### SPEC-034 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-003 (auto-loop resolves spec gaps autonomously)
- **File**: .github/agents/planner.agent.md#L103-L131
- **Description**: The auto-loop mechanism is fully implemented in Step 3: gap report creation, Spec Architect invocation via `runSubagent`, retry logic (up to 3 attempts), re-read and re-check after each attempt, and human escalation after exhausting retries. The mechanism satisfies SC-003's verification criteria.

### SPEC-035 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-004 (each skill runs in fresh context window)
- **File**: .github/agents/planner.agent.md#L205-L261
- **Description**: Each skill is dispatched via `runSubagent` (Steps 8 and 9), which by definition runs in a fresh context. Each prompt instructs the skill to "Read the skill instructions at: <skill_path>" as its first action, confirming the skill loads only its own SKILL.md.

### SPEC-036 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-005 (cross-WP consistency validated before presenting to user)
- **File**: .github/agents/planner.agent.md#L262-L285
- **Description**: Step 10 runs the full cross-WP consistency audit (7 checks) and WP implementation-completeness check (5 checks) before Step 11 presents to the user. Satisfies SC-005.

### SPEC-037 [PASS]
- **Checklist item**: Success criteria verification
- **Requirement**: SC-006 (adding new planning dimension requires only one skill file)
- **File**: .github/agents/planner.agent.md#L181-L203
- **Description**: Dynamic skill discovery in Step 7 uses glob pattern `.github/skills/plan-*/SKILL.md`. Unknown skills are dispatched after canonical skills in alphabetical order. No coordinator edit is needed to add a new planning dimension. Satisfies SC-006.
