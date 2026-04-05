---
lane: for_review
---

# WP15 - Planner Coordinator Rewrite

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/003-planner-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP14 |
| Goal | Refactor `planner.agent.md` from monolithic V1 to skill-based V2 coordinator with two-phase execution, dynamic skill discovery, auto-loop gap resolution, and post-completion validation |
| Status | Complete |
| Independent Test | Invoke the Planner with a validated spec. Verify: it lists specs, validates status, runs completeness pre-check, discovers 8 plan skills, dispatches Phase 1 then Phase 2, runs post-completion validation, presents plan, and commits |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP15-planner-coordinator.md` |

## Objective

Rewrite `.github/agents/planner.agent.md` from the current monolithic single-pass planner into a lightweight coordinator that dispatches 8 sequential planning skills across two phases. The coordinator handles the full lifecycle: spec selection and validation, completeness pre-check with auto-loop to Spec Architect, workspace and web research, patterns consumption, plan accumulator initialization, dynamic skill discovery, Phase 1 decomposition dispatch, Phase 2 contract generation dispatch, post-completion validation, user presentation, and commit. It writes no plan content itself beyond the skeleton README.

## Spec References

FR-001 through FR-022, Section 8.1 through 8.5, Section 9.1, Section 9.4

## Tasks

### T15-01 - Define YAML frontmatter and tool declarations

- **Description**: Write the YAML frontmatter for the planner coordinator agent file. Include `name`, `description`, `tools` (runSubagent, vscode_askQuestions, file system, terminal, web, manage_todo_list), and handoff buttons (Start Implementation -> Coder, Clarify Specification -> Spec Architect).
- **Spec refs**: FR-020, FR-021, Section 8.1, Section 8.5
- **Parallel**: No
- **Acceptance criteria**:
  - [x] `name` is "3. Planner"
  - [x] `tools` list includes `runSubagent` for skill dispatch
  - [x] `tools` list includes `vscode_askQuestions` for spec selection and alignment
  - [x] Handoff buttons defined: "Start Implementation" (Coder) and "Clarify Specification" (Spec Architect)
  - [x] YAML is valid and frontmatter starts on line 1
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Mirror `.github/agents/spec-architect.agent.md` frontmatter structure
  - Official docs: https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview
  - Known pitfalls: Agent name must match the expected handoff target from other agents (Orchestrator, Spec Architect)

### T15-02 - Implement spec selection and status validation (Steps 1-2)

- **Description**: Write the coordinator workflow for listing specs in `.sdd/specs/`, presenting selection to user (or confirming single spec), reading the full spec + companion artifacts, and validating the spec's Status field is "Validated" or "Final".
- **Spec refs**: FR-001, FR-002, FR-003
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL list all files in `.sdd/specs/` and present them to the user for selection (FR-001)
  - [x] If only one spec exists, the coordinator SHALL confirm it before proceeding (FR-001)
  - [x] If `.sdd/specs/` is empty, the coordinator SHALL inform the user and halt (FR-001 error)
  - [x] The coordinator SHALL read the selected spec in full, including companion artifacts in `.sdd/specs/artifacts/<NNN>-<idea-name>/` (FR-002)
  - [x] The coordinator SHALL verify the spec's Status is "Validated" or "Final"; if "Draft", refuse and recommend Spec Architect (FR-003)
- **Test requirements**: BDD (Scenario 8: spec status validation)
- **Depends on**: T15-01
- **Implementation Guidance**:
  - Pattern: Use `list_dir` to scan `.sdd/specs/`, `vscode_askQuestions` for selection
  - Error handling: Empty directory -> halt with message. Draft status -> halt with Spec Architect recommendation.
  - Spec companion artifacts path: `.sdd/specs/artifacts/<NNN>-<idea-name>/`

### T15-03 - Implement spec completeness pre-check (Step 3)

- **Description**: Write the 7-point completeness pre-check against the spec (FR-004) and artifact consistency verification (FR-005). If any check fails, create a structured gap report.
- **Spec refs**: FR-004, FR-005, Section 7.3 (Gap Report data model)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL verify: traceability matrix has no empty cells (FR-004.1)
  - [x] The coordinator SHALL verify: every FR has defined error behavior (FR-004.2)
  - [x] The coordinator SHALL verify: every entity field has type, constraints, and validation rules (FR-004.3)
  - [x] The coordinator SHALL verify: every API endpoint has all applicable error codes (FR-004.4)
  - [x] The coordinator SHALL verify: every external integration has timeout/retry/fallback (FR-004.5)
  - [x] The coordinator SHALL verify: every stateful entity has explicit state transitions (FR-004.6)
  - [x] The coordinator SHALL verify: cross-cutting concerns are addressed (FR-004.7)
  - [x] The coordinator SHALL verify spec companion artifacts are consistent with the prose spec (FR-005)
  - [x] Gap report uses the data model from Section 7.3: Gap ID, Category, FR reference, Description, Impact
- **Test requirements**: BDD (Scenario 2: auto-loop resolves gaps)
- **Depends on**: T15-02
- **Implementation Guidance**:
  - Gap report categories (from Section 7.3): traceability, error-behavior, data-validation, api-errors, integration-failure, state-machine, cross-cutting
  - The gap report is markdown-formatted and passed to Spec Architect in auto-loop

### T15-04 - Implement auto-loop to Spec Architect (Step 4)

- **Description**: Write the auto-loop logic: when spec gaps are discovered, invoke Spec Architect via `runSubagent` with the gap report, spec path, and artifacts directory. Retry up to 3 times. After 3 failures, escalate to human.
- **Spec refs**: FR-006, Section 8.4 (Auto-Loop Prompt Template)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL invoke Spec Architect via `runSubagent` with gap report, spec path, artifacts dir (FR-006)
  - [x] The coordinator SHALL retry up to 3 times if gaps remain after each attempt (FR-006)
  - [x] After 3 failed iterations, the coordinator SHALL escalate to the human with the full gap report (FR-006)
  - [x] On Spec Architect subagent failure, the coordinator SHALL escalate to human with full context (FR-006 error)
  - [x] After successful auto-loop, the coordinator SHALL re-read the spec and re-run the completeness pre-check (FR-006 postcondition)
  - [x] Auto-loop prompt SHALL match Section 8.4 template
- **Test requirements**: BDD (Scenario 2: auto-loop on first attempt, Scenario 3: escalation after 3 failures)
- **Depends on**: T15-03
- **Implementation Guidance**:
  - Auto-loop prompt template from Section 8.4:
    ```
    Spec gaps discovered during planning decomposition.
    Spec: <spec_path>
    Companion artifacts: <spec_artifacts_dir>
    Gap Report: <gap_report_markdown>
    This is auto-loop attempt <N> of 3.
    ```
  - Known pitfalls: Must re-read spec after each successful loop (spec file may have changed on disk)

### T15-05 - Implement research phase (Step 5)

- **Description**: Write the research step: invoke workspace exploration subagent (FR-007) and conduct web research (FR-008). The research subagent SHALL NOT draft plan content.
- **Spec refs**: FR-007, FR-008
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL invoke a workspace research subagent to discover existing code, structure, build system, test frameworks, patterns, and existing plans (FR-007)
  - [x] The research subagent SHALL NOT draft any plan content (FR-007)
  - [x] The coordinator SHALL conduct web research for official docs, known gotchas, testing framework guides, and starter templates (FR-008)
- **Test requirements**: none
- **Depends on**: T15-02
- **Implementation Guidance**:
  - Pattern: Use `runSubagent` with agent "Explore" for workspace research
  - Web research: Use `fetch_webpage` for official docs of libraries in the spec's tech stack
  - The research summary is passed to each skill dispatch as `research_summary` input

### T15-06 - Implement patterns consumption and plan initialization (Steps 6-7)

- **Description**: Write the patterns consumption step (FR-009) and plan accumulator initialization step (FR-017). Read `plan-patterns.md` if it exists, create plan and contracts directories, and write skeleton README.
- **Spec refs**: FR-009, FR-017
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL read `.sdd/reviews/plan-patterns.md` if it exists and include active patterns in skill prompts (FR-009)
  - [x] The coordinator SHALL create `.sdd/plans/` if it does not exist (FR-017.1)
  - [x] The coordinator SHALL create `.sdd/plans/contracts/` if it does not exist (FR-017.2)
  - [x] The coordinator SHALL write a skeleton README at `.sdd/plans/README.md` with spec reference, target language, and plan status "In Progress" (FR-017.3)
- **Test requirements**: none
- **Depends on**: T15-05
- **Implementation Guidance**:
  - If `plan-patterns.md` does not exist, proceed without patterns (empty string for patterns input)
  - Skeleton README should be minimal -- the plan-acceptance skill (Phase 1) populates it fully in FR-036

### T15-07 - Implement dynamic skill discovery (Step 8)

- **Description**: Write the skill discovery logic: scan for directories matching `.github/skills/plan-*/SKILL.md`, sort them into canonical order (FR-011), and handle zero-skills error.
- **Spec refs**: FR-010, FR-011
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL discover planning skills by scanning `.github/skills/plan-*/SKILL.md` (FR-010)
  - [x] A sorted list of discovered skill names SHALL be produced (FR-010 postcondition)
  - [x] If zero skills are discovered, the coordinator SHALL halt and report the error (FR-010 error)
  - [x] Skills SHALL be dispatched in canonical order: plan-decomposition, plan-acceptance, plan-interface-contracts, plan-data-schemas, plan-api-contracts, plan-state-machines, plan-error-catalogs, plan-cross-wp-validation (FR-011)
  - [x] Skills present but not in the canonical list SHALL be dispatched after all known skills, in alphabetical order (FR-011)
  - [x] Skills not present SHALL be skipped without error (FR-011)
- **Test requirements**: BDD (Scenario 7: dynamic skill discovery)
- **Depends on**: T15-06
- **Implementation Guidance**:
  - Use `file_search` with glob `.github/skills/plan-*/SKILL.md` for discovery
  - Canonical order is hardcoded as a list; discovered skills are sorted against it
  - Unknown skills go alphabetically after the canonical list

### T15-08 - Implement Phase 1 and Phase 2 skill dispatch (Steps 9-10)

- **Description**: Write the two-phase dispatch logic. Phase 1 dispatches `plan-decomposition` then `plan-acceptance` sequentially via `runSubagent`. Phase 2 dispatches the remaining 6 skills sequentially. Phase 1 failures halt immediately; Phase 2 failures are logged and skipped.
- **Spec refs**: FR-012, FR-013, FR-014, FR-015, FR-016, Section 8.2, Section 8.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Phase 1 skills execute sequentially, blocking (FR-012, FR-015)
  - [x] Phase 2 skills execute sequentially, skipping failed skills (FR-012, FR-015)
  - [x] Each skill dispatch includes all 9 inputs from FR-023 via the prompt template (FR-014)
  - [x] Phase 1 prompt matches Section 8.2 template (FR-014)
  - [x] Phase 2 prompt matches Section 8.3 template (FR-014)
  - [x] Phase 1 failure halts immediately (FR-014 error)
  - [x] Phase 2 failure logs error and continues to next skill (FR-014 error)
  - [x] Each skill reads current plan state before writing (FR-016)
  - [x] Phase 2 contracts generated in 800-line blocks per WP (FR-013)
- **Test requirements**: BDD (Scenario 1: full plan generation, Scenario 6: Phase 2 failure tolerance)
- **Depends on**: T15-07
- **Implementation Guidance**:
  - Phase 1 template (Section 8.2): 7 numbered steps + rules block
  - Phase 2 template (Section 8.3): 5 numbered steps + rules block
  - The 800-line block limit (FR-013) is enforced by Phase 2 skills themselves, not by the coordinator
  - Known pitfalls: Each skill must receive the correct phase indicator (1 or 2)

### T15-09 - Implement post-completion validation (Step 11)

- **Description**: Write the post-completion validation: cross-WP consistency audit (FR-018) and WP implementation-completeness check (FR-019). Fix any issues found inline and document in README.
- **Spec refs**: FR-018, FR-019
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL verify data contract consistency across WPs (FR-018.1)
  - [x] The coordinator SHALL verify API/interface contract consistency (FR-018.2)
  - [x] The coordinator SHALL verify dependency integrity with no circular dependencies (FR-018.3)
  - [x] The coordinator SHALL verify configuration consistency (FR-018.4)
  - [x] The coordinator SHALL verify test consistency at 80% code, 90% branch (FR-018.5)
  - [x] The coordinator SHALL verify spec traceability: every FR assigned to exactly one task (FR-018.6)
  - [x] The coordinator SHALL verify contract files match WP task specs (FR-018.7)
  - [x] The coordinator SHALL verify each WP has 5-12 tasks (FR-019.1)
  - [x] The coordinator SHALL verify at least 3 acceptance criteria per task (FR-019.2)
  - [x] The coordinator SHALL verify implementation guidance with doc links per task (FR-019.3)
  - [x] The coordinator SHALL verify contract file references per task (FR-019.4)
  - [x] The coordinator SHALL verify no ambiguous language (FR-019.5)
  - [x] Inconsistencies found SHALL be fixed and documented in README "Consistency Notes" (FR-018)
- **Test requirements**: BDD (Scenario 4: cross-WP validation catches inconsistency)
- **Depends on**: T15-08
- **Implementation Guidance**:
  - This is the coordinator's own validation step, separate from the plan-cross-wp-validation skill (FR-051)
  - The skill handles contract-level consistency; the coordinator validates the overall plan structure
  - Known pitfalls: Must read all WP files and contract files to perform cross-WP checks

### T15-10 - Implement presentation, approval, and commit (Steps 12-13)

- **Description**: Write the presentation step (show plan in chat), approval handling (revise/clarify/approve), and commit policy (individual WP commits, per-WP contract commits, standalone README commit).
- **Spec refs**: FR-020, FR-021, FR-022, Section 8.5
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL present the plan to the user in chat, not just as files (FR-020)
  - [x] On change requests, the coordinator SHALL revise WPs and re-validate (FR-021)
  - [x] On questions, the coordinator SHALL clarify or ask follow-ups (FR-021)
  - [x] On approval, the coordinator SHALL acknowledge and recommend Coder for WP01 (FR-021)
  - [x] Each WP file SHALL be committed individually with `git add` listing files explicitly (FR-022)
  - [x] README SHALL be committed as a standalone change (FR-022)
  - [x] Contract files SHALL be committed per-WP (FR-022)
  - [x] Handoff prompt to Coder SHALL match Section 8.5 template (FR-021)
- **Test requirements**: BDD (Scenario 1: full plan generation)
- **Depends on**: T15-09
- **Implementation Guidance**:
  - Commit message format: `docs(plan): add WP<NN> <title>` for WP files, `docs(plan): add contracts for WP<NN>` for contracts
  - Always use explicit `git add` with listed files, never `git add .`

### T15-11 - Verify encoding and coordinator line count

- **Description**: Verify the completed coordinator has no prohibited Unicode characters and stays within a reasonable line count (target 300-500 lines, matching the Spec Architect V2 coordinator pattern).
- **Spec refs**: Section 9.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] No em dashes, smart quotes, or curly apostrophes in the coordinator file
  - [x] Coordinator line count is between 300-500 lines (target established by Spec Architect V2 at 365 lines)
  - [x] All hyphens are ASCII `-` (U+002D)
- **Test requirements**: unit (encoding validation)
- **Depends on**: T15-10
- **Implementation Guidance**:
  - Reuse the Python encoding check from WP14
  - The 300-500 line target is a guideline, not a hard constraint; quality over brevity

## Implementation Notes

- The Planner V2 coordinator mirrors the Spec Architect V2 coordinator pattern (WP09)
- Key structural difference: Planner has two-phase dispatch (Phase 1 decomposition, Phase 2 contracts) whereas Spec Architect has single-phase
- The auto-loop to Spec Architect (FR-006) is unique to the Planner -- no other coordinator has this pattern

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T10:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T11:00:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T18:00:00Z - review-coordinator - lane=to_do - Verdict: Changes Required (2 FAILs) -- awaiting remediation

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T18:00:00Z
> **Verdict**: Changes Required
> **Skills dispatched**: review-spec (FAIL), review-quality (WARN)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked
- [PASS] Activity Log: Consistent lane transitions
- [WARN] Commit granularity: Single bulk commit (9036def) for all 11 tasks
- [PASS] Encoding: No violations found

### Review Feedback

> Implementers: address every FB-XX item before returning for re-review.

- [ ] **FB-01**: [spec-adherence] FR-008 item 4 deviation - Implementation says "CI/CD best practices for the target platform" instead of spec's "Starter templates or boilerplate repos matching the tech stack".
  File: .github/agents/planner.agent.md#L159. Expected: Change to "Starter templates or boilerplate repos matching the tech stack" per FR-008.
  Source skills: review-spec (SPEC-013)
- [ ] **FB-02**: [spec-adherence] FR-019 ambiguous language list incomplete - Omits "should" from the banned terms list. Spec says ("should", "appropriate", "reasonable") but implementation says ("appropriate", "reasonable", "as needed", "etc.", "similar").
  File: .github/agents/planner.agent.md#L282. Expected: Add "should" to the list. Keep the extra terms.
  Source skills: review-spec (SPEC-025)

### Warnings
- [WARN] SPEC-020: FR-014 Phase 2 prompt omits research_summary and patterns inputs, but matches spec's own Section 8.3 template verbatim. Spec-internal inconsistency. (review-spec SPEC-020)
- [WARN] QUAL-001: Orphaned closing `</plan_templates>` tag with no matching opening tag. (review-quality QUAL-001)
- [WARN] QUAL-002: Uses `fetch_webpage` in text while spec-architect uses `web/fetch`. (review-quality QUAL-002)
- [WARN] PROC-003: Single bulk commit for all tasks.

### Cross-Correlation Notes
- SPEC-020 (FR-014 inputs) may be reclassified pending spec clarification -- the implementation follows Section 8.3 verbatim.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 2 | 2 | 0 |
| review-spec | 29 | 1 | 2 |
| review-quality | 6 | 2 | 0 |
| **Total** | **37** | **5** | **2** |
- The coordinator file replaces the existing V1 planner at `.github/agents/planner.agent.md`

## Parallel Opportunities

- T15-05 (research) can start after T15-02 completes, in parallel with T15-03/T15-04 (pre-check and auto-loop) if the pre-check passes
- In practice, most tasks are sequential due to the coordinator's linear workflow

## Risks & Mitigations

- **Risk**: Auto-loop to Spec Architect may not work if the Spec Architect agent does not support being invoked as a subagent for targeted gap fixes. **Mitigation**: The Spec Architect V2 (WP09) is already implemented and supports subagent invocation.
- **Risk**: Two-phase dispatch complexity increases coordinator line count beyond target. **Mitigation**: Extract repeated dispatch logic into a template string; keep the coordinator as a workflow orchestrator, not a content producer.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T10:20:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T10:45:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T19:00:00Z - coder - lane=for_review - Remediated FB-01 (FR-008 text), FB-02 (FR-019 ambiguous terms), QUAL-001 (orphaned tag)
