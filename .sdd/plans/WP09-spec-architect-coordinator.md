---
lane: done
---

# WP09 - Spec Architect Coordinator

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/002-spec-architect-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP08 |
| Goal | Refactor spec-architect.agent.md from monolithic V1 to skill-based coordinator V2 |
| Status | Complete |
| Independent Test | Invoke the Spec Architect with a test brief. Verify: it lists briefs, conducts gap analysis, initializes the accumulator, discovers 8 skills, dispatches them sequentially, runs post-completion validation, and commits |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP09-spec-architect-coordinator.md` |

## Objective

Rewrite `.github/agents/spec-architect.agent.md` from the current monolithic single-pass spec generator into a lightweight coordinator that dispatches 8 sequential skills. The coordinator handles the lifecycle: brief selection, research, gap analysis, accumulator initialization, dynamic skill discovery, sequential dispatch with context forwarding, post-completion validation, artifact consistency checking, user presentation, and commit. It writes no spec sections itself (except sections 1-3).

## Spec References

- FR-001 through FR-022 (entire coordinator subsystem)
- Section 8.2 (skill subagent prompt template)
- Section 8.3 (handoff prompt templates)
- Section 9.1 (system design interaction pattern)
- Section 9.4 (key design decisions)

## Tasks

### T09-01 - Refactor agent file YAML frontmatter

- **Description**: Update `.github/agents/spec-architect.agent.md` YAML frontmatter to reflect the V2 coordinator role. Update name, description (with trigger keywords), model, tools list (must include `agent/runSubagent` for skill dispatch), and handoffs (to Planner, to Ideation).
- **Spec refs**: Section 7.3, Section 8.1
- **Parallel**: No (all other tasks build on this)
- **Acceptance criteria**:
  - [x] `name` field is `"2. Spec Architect"`
  - [x] `description` includes trigger keywords: "write spec, create specification, spec this out, architect this, turn brief into spec"
  - [x] `model` is `Claude Opus 4.6 (copilot)`
  - [x] `tools` array includes: `agent/runSubagent`, `read/readFile`, `edit/createFile`, `edit/editFiles`, `search/fileSearch`, `search/textSearch`, `search/codebase`, `search/listDirectory`, `web/fetch`, `vscode/askQuestions`, `todo`
  - [x] `handoffs` includes: "Create Plan" to Planner, "Return to Ideation" to Ideation
  - [x] `argument-hint` provides usage guidance
- **Test requirements**: none (YAML validation)
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: existing `.github/agents/review-coordinator.agent.md` for V2 YAML pattern
  - The V1 agent file already exists -- this is a modification, not a creation
  - Keep the V1 content available for reference but rewrite the body entirely
  - Handoff templates from Section 8.3:
    - To Planner: "Specification {spec_path} has been validated and approved. Companion artifacts are at: {artifacts_dir}. Please decompose into work packages with contracts."
    - To Ideation: "The specification process has identified fundamental issues with the brief. Issues: {list}. Please revise the ideation brief."

### T09-02 - Write brief selection logic

- **Description**: Write the coordinator section that handles brief selection from `.sdd/ideas/`. This covers listing briefs, presenting selection to user (or confirming if only one), reading the selected brief in full, and handling error cases.
- **Spec refs**: FR-001, FR-002
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator lists all `.md` files in `.sdd/ideas/`
  - [x] If multiple briefs exist, coordinator presents selection via `vscode_askQuestions`
  - [x] If only one brief exists, coordinator confirms it before proceeding
  - [x] If `.sdd/ideas/` is empty, coordinator halts with informative message
  - [x] If selected brief is unreadable, coordinator halts with filesystem error
  - [x] Selected brief is read in full before any subsequent step
- **Test requirements**: BDD (Scenario 1, Scenario 3 from Section 11.2)
- **Depends on**: T09-01
- **Implementation Guidance**:
  - Use `list_dir` to scan `.sdd/ideas/`
  - Use `vscode_askQuestions` for selection (reference: existing planner agent pattern)
  - Use `read_file` to load the selected brief
  - Error messages should be actionable: "No briefs found in .sdd/ideas/. Create an ideation brief first using the Ideation agent."
  - Pattern reference: review-coordinator.agent.md scope selection logic

### T09-03 - Write research phase instructions

- **Description**: Write the coordinator section that handles workspace research and web research. The coordinator invokes a subagent for workspace discovery and performs web research (competitors, tech versions, OWASP, standards) before any spec writing.
- **Spec refs**: FR-003, FR-004
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator dispatches a workspace research subagent using `runSubagent` to discover existing code, patterns, and constraints (FR-003)
  - [x] Research subagent is instructed to perform discovery only -- no spec drafting
  - [x] Coordinator conducts web research for: competing projects, tech versions, OWASP entries, relevant standards/RFCs (FR-004)
  - [x] Research findings are summarized and stored for passing to each skill
- **Test requirements**: BDD (Scenario 1 from Section 11.2)
- **Depends on**: T09-02
- **Implementation Guidance**:
  - Workspace research subagent prompt: "Search the workspace for existing code, configuration, and documentation related to {brief topic}. Identify patterns, frameworks, and constraints. Do NOT draft any spec content."
  - Use the `Explore` agent for workspace research (it supports thoroughness levels)
  - Web research targets from FR-004:
    1. Competing/analogous open-source projects
    2. Latest stable versions of all technologies mentioned
    3. Known pitfalls and anti-patterns for the tech stack
    4. OWASP entries for the system's threat surface
    5. Relevant standards or RFCs
  - Use `fetch_webpage` for web research

### T09-04 - Write gap analysis flow

- **Description**: Write the coordinator section that identifies and resolves gaps in the brief before dispatching skills. Gaps are categorized (functional, data, architecture, non-functional, testing) and tracked. Questions are asked in batches of max 3 per turn.
- **Spec refs**: FR-005, FR-006
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator identifies gaps and categorizes them: Functional, Data & Domain, Architecture & Technology, Non-Functional, Testing
  - [x] Gaps are tracked using `manage_todo_list`
  - [x] Questions are asked via `vscode_askQuestions` in batches of max 3 per turn
  - [x] If answers significantly change scope, coordinator loops back to research phase
  - [x] Coordinator does NOT proceed to skill dispatch until all critical gaps are resolved
  - [x] Gap resolution is documented
- **Test requirements**: BDD (Scenario 3, Scenario 6 from Section 11.2)
- **Depends on**: T09-03
- **Implementation Guidance**:
  - Gap categories from FR-005:
    - **Functional**: user flows, edge cases, error states, actor permissions
    - **Data & Domain**: entities, attributes, relationships, validation rules
    - **Architecture & Technology**: platform, stack, integrations, deployment
    - **Non-Functional**: performance, security, scalability, accessibility
    - **Testing**: critical behaviors to verify, compliance obligations
  - Max 3 questions per `vscode_askQuestions` call (FR-006)
  - Loop condition: if user's answer reveals a new technology or fundamentally different architecture, re-run research

### T09-05 - Write accumulator initialization

- **Description**: Write the coordinator section that creates the accumulator file (the spec itself) with sections 1-3 and creates the companion artifacts directory.
- **Spec refs**: FR-007, FR-008
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator determines file number `<NNN>` by checking existing files in `.sdd/specs/` and incrementing
  - [x] Accumulator file created at `.sdd/specs/<NNN>-<idea-name>.spec.md`
  - [x] File contains spec header (title, source brief path, feature branch, status=Draft, version=1.0)
  - [x] File contains Section 1 (Overview) written by coordinator from brief + research
  - [x] File contains Section 2 (Goals & Success Criteria) derived from brief's vision
  - [x] File contains Section 3 (Users & Roles) derived from brief's target users
  - [x] Companion artifacts directory created at `.sdd/specs/artifacts/<NNN>-<idea-name>/`
- **Test requirements**: BDD (Scenario 1 from Section 11.2)
- **Depends on**: T09-04
- **Implementation Guidance**:
  - Use `list_dir` on `.sdd/specs/` to find highest numbered spec, then increment
  - Naming: extract slug from brief filename (e.g., `pipeline-v2` from `002-sdd-pipeline-v2.md`)
  - Sections 1-3 are the ONLY sections the coordinator writes directly; all others come from skills
  - Use `create_file` for the accumulator; use `create_directory` (or mkdir via terminal) for artifacts dir
  - Accumulator header template:
    ```markdown
    # <Title> -- Specification

    > **Source brief**: `.sdd/ideas/<brief-file>`
    > **Feature branch**: `<NNN>-<slug>`
    > **Status**: Draft
    > **Version**: 1.0
    ```

### T09-06 - Write dynamic skill discovery

- **Description**: Write the coordinator section that discovers available spec skills by scanning `.github/skills/spec-*/SKILL.md` and orders them per the canonical sequence.
- **Spec refs**: FR-009, FR-010
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator scans for `.github/skills/spec-*/SKILL.md` using glob/file_search
  - [x] Zero skills discovered: coordinator halts with "No spec skills are installed"
  - [x] Discovered skills are sorted into canonical order (FR-010): requirements, user-stories, data-model, api-design, architecture, security, test-strategy, traceability
  - [x] Skills present but NOT in the canonical list are dispatched AFTER all known skills, in alphabetical order
  - [x] Skills from the canonical list that are NOT present are skipped without error
- **Test requirements**: BDD (Scenario 7 from Section 11.2)
- **Depends on**: T09-05
- **Implementation Guidance**:
  - Use `file_search` with glob `.github/skills/spec-*/SKILL.md` to discover skills
  - Canonical order array (hardcoded in coordinator):
    ```
    ["spec-requirements", "spec-user-stories", "spec-data-model", "spec-api-design",
     "spec-architecture", "spec-security", "spec-test-strategy", "spec-traceability"]
    ```
  - Sorting logic: known skills go first in canonical order; unknown skills go after in alphabetical order
  - This enables extensibility (SC-005): a new `spec-compliance` skill is auto-discovered

### T09-07 - Write skill dispatch loop

- **Description**: Write the coordinator section that dispatches each discovered skill sequentially as a subagent, passing the accumulator path, artifacts directory, brief, research summary, section numbers, patterns, and target language (FR-011 through FR-013).
- **Spec refs**: FR-011, FR-012, FR-013, Section 8.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each skill dispatched via `runSubagent` with all 8 input parameters from FR-023
  - [x] Skills dispatched one at a time, blocking until each returns (FR-012)
  - [x] Each skill prompt includes instruction to read accumulator first (FR-013 context forwarding)
  - [x] Prompt follows the template from Section 8.2
  - [x] If a skill subagent fails, coordinator halts immediately with error report (FR-011 error behavior)
  - [x] Coordinator does NOT dispatch next skill until current one returns successfully
  - [x] Section number assignments per skill match the spec: requirements=4,10,12,13; user-stories=5,6; data-model=7; api-design=8; architecture=9; security=10.2; test-strategy=11; traceability=14,15,16,17,18
- **Test requirements**: BDD (Scenarios 1, 2, 5 from Section 11.2)
- **Depends on**: T09-06
- **Implementation Guidance**:
  - Skill prompt template from Section 8.2 (copy verbatim into coordinator):
    ```
    Write Section <section_numbers> of the specification.

    1. Read the skill instructions at: <skill_path>
    2. Read the current spec state at: <accumulator_path>
    3. Read the source brief at: <brief_path>
    4. Research context: <research_summary>
    5. Active patterns to avoid: <patterns>
    6. Target language for artifacts: <target_language>
    7. Artifacts directory: <artifacts_dir>

    Write your section(s) to the spec file at <accumulator_path> by APPENDING after the existing content.
    If this skill produces companion artifacts, write them to <artifacts_dir>.
    ```
  - On fail: "Skill <name> failed. The spec cannot be completed without Section <N>. Error: <details>"
  - Design decision reference: Section 9.4 Decision 2 (skills halt on failure, unlike Reviewer)

### T09-08 - Write companion artifact management instructions

- **Description**: Write the coordinator section that instructs skills on companion artifact production: file naming, target language selection, manifest comments, and artifact types.
- **Spec refs**: FR-014, FR-015, FR-016, FR-028
- **Parallel**: Yes (can be developed alongside T09-07)
- **Acceptance criteria**:
  - [x] Coordinator determines target language from brief/spec tech stack section
  - [x] If target language not specified, default to TypeScript (FR-015)
  - [x] Artifact file naming convention documented in coordinator: `data-models.<ext>`, `state-machines.<ext>`, `api-contracts.<ext>`, `error-catalog.<ext>`, `interfaces.<ext>`, `config-schema.<ext>`
  - [x] Manifest comment template included in each skill's prompt (FR-028):
    ```
    // Generated by: spec-<skill-name> skill
    // Source spec: .sdd/specs/<NNN>-<idea-name>.spec.md, Section <N>
    // Target language: <language>
    // DO NOT EDIT MANUALLY -- regenerated on spec revision
    ```
  - [x] Artifact types from FR-014 are listed: data model schemas, API contracts, interface definitions, state machine definitions, error catalogs, configuration schemas
- **Test requirements**: BDD (Scenario 1 from Section 11.2 -- artifacts exist after generation)
- **Depends on**: T09-01
- **Implementation Guidance**:
  - File extensions by language: TypeScript=`.ts`, Python=`.py`, SQL=`.sql`
  - Artifacts contain TYPE DEFINITIONS ONLY (NFR-005): no I/O, no network, no filesystem operations
  - The coordinator passes `target_language` as part of every skill prompt
  - If brief mentions "Python project", target_language = "Python", ext = ".py"

### T09-09 - Write post-completion validation

- **Description**: Write the coordinator section that validates the completed spec after all skills finish. This covers the 10-point checklist (FR-017) and artifact consistency checks (FR-018).
- **Spec refs**: FR-017, FR-018
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator checks all 10 validation points from FR-017:
    1. Every FR uses SHALL or SHALL NOT
    2. Every FR has error behavior defined
    3. Every entity has all fields with types, constraints, validation
    4. Every API endpoint has all applicable response codes
    5. Every external integration has failure strategy
    6. State transitions are explicit for entities with status fields
    7. Traceability matrix has no empty cells
    8. No ambiguous words: "appropriate", "reasonable", "as needed", "etc.", "similar"
    9. All `[NEEDS CLARIFICATION]` markers resolved
    10. Encoding compliance: no em dashes, smart quotes, curly apostrophes
  - [x] Coordinator checks artifact consistency (FR-018):
    1. Field names in data model artifacts match Section 7
    2. Endpoint signatures in API artifacts match Section 8
    3. Error codes in error catalog match Section 4
    4. State values in state machine artifacts match Section 7
  - [x] Validation failures are fixed inline or flagged to user
  - [x] Coordinator resolves all `[CROSS-REF ISSUE]` markers
- **Test requirements**: BDD (Scenarios 3, 4 from Section 11.2)
- **Depends on**: T09-07
- **Implementation Guidance**:
  - For text checks (SHALL, ambiguous words, encoding): use `grep_search` on the accumulator file
  - For artifact consistency: use `read_file` on each artifact and cross-reference with the prose sections
  - CROSS-REF ISSUE resolution: read the marker, determine which section's definition is authoritative, update the other
  - Encoding check: search for unicode characters U+2013 (en dash), U+2014 (em dash), U+201C/U+201D (smart quotes), U+2018/U+2019 (curly apostrophes)

### T09-10 - Write patterns consumption

- **Description**: Write the coordinator section that reads `.sdd/reviews/spec-patterns.md` before dispatching skills and includes active pattern summaries in each skill's prompt.
- **Spec refs**: FR-019
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Coordinator reads `.sdd/reviews/spec-patterns.md` if it exists
  - [x] If file does not exist, coordinator continues without patterns (no error)
  - [x] Active patterns are extracted and included in each skill's dispatch prompt
  - [x] Pattern summaries are concise enough to not bloat skill prompts
- **Test requirements**: BDD (Scenario 1 from Section 11.2)
- **Depends on**: T09-01
- **Implementation Guidance**:
  - Use `read_file` on `.sdd/reviews/spec-patterns.md`
  - Parse the "Active Patterns" section for current patterns
  - If no active patterns, pass "No active patterns" to skills
  - Patterns inform skills about known mistakes to avoid (e.g., "PAT-005: Missing error behavior on FR definitions")
  - Keep pattern summaries to 1-2 lines each to conserve context window

### T09-11 - Write presentation, approval, and commit flow

- **Description**: Write the coordinator section that presents the completed spec to the user, handles feedback (approval, changes, questions, new requirements), sets status to Validated on approval, and commits all files.
- **Spec refs**: FR-020, FR-021, FR-022, Section 8.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator presents the spec content in chat after validation (FR-020)
  - [x] On approval: coordinator changes status from "Draft" to "Validated" in the spec file
  - [x] On approval: coordinator commits spec file + all artifact files with explicit `git add`
  - [x] Commit message format: `docs(spec): add <idea name> specification v1.0`
  - [x] On revision request: coordinator re-dispatches affected skills or edits inline, re-validates, re-presents
  - [x] On questions: coordinator clarifies or asks follow-up questions
  - [x] On new requirements: coordinator loops back to research and gap analysis
  - [x] Revision commit message: `docs(spec): revise <idea name> spec -- <brief description>`
  - [x] Files listed explicitly in `git add` (never `git add .` or `git add -A`)
- **Test requirements**: BDD (Scenario 1, Scenario 8 from Section 11.2)
- **Depends on**: T09-09
- **Implementation Guidance**:
  - Status change: use `replace_string_in_file` to change "Draft" to "Validated" in the spec header
  - Commit command: `git add .sdd/specs/<NNN>-<name>.spec.md .sdd/specs/artifacts/<NNN>-<name>/* ; git commit -m "docs(spec): add <name> specification v1.0"`
  - Revision support (Spec Revision Flow from Section 6.2): identify affected sections, re-dispatch only those skills plus downstream dependents
  - Handoff buttons from Section 8.3: "Create Plan" sends to Planner with spec path and artifacts dir

## Implementation Notes

- The coordinator agent file (`.github/agents/spec-architect.agent.md`) will be substantially rewritten. The V1 content is being replaced, not extended.
- The coordinator is a markdown file with embedded logic in prose instructions. There is no executable code.
- The coordinator relies on `runSubagent` for skill dispatch. Each subagent gets a fresh context window (SC-001).
- The total coordinator file should be 300-500 lines, consistent with the existing review-coordinator.agent.md.
- Keep the coordinator focused on ORCHESTRATION. All section-writing logic belongs in the skills (WP10-WP13).

## Parallel Opportunities

- T09-08 (artifact management) and T09-10 (patterns consumption) can be developed in parallel with T09-07 (dispatch loop) since they are independent concerns injected into the prompt.
- T09-01 (frontmatter) must be done first.
- T09-02 through T09-07 are sequential (each extends the coordinator flow).

## Risks & Mitigations

- **Risk**: Coordinator becomes too large for a single .agent.md file (context limit).
  - **Mitigation**: Keep instructions concise. Delegate ALL section-writing to skills. Target 300-500 lines.
- **Risk**: Skill dispatch prompt template doesn't provide enough context for skills to write quality sections.
  - **Mitigation**: Template from Section 8.2 was designed with 8 inputs including accumulator path and research summary.
- **Risk**: Post-completion validation misses inconsistencies between prose and artifacts.
  - **Mitigation**: Systematic field-by-field cross-reference (FR-018). Automated grep for known issues (FR-017).

## Self-Review

**Spec Compliance**: All 11 tasks implemented. All acceptance criteria (59 total) checked off. Every FR from FR-001 through FR-022 plus FR-028 is covered in the coordinator workflow steps.

**Correctness**: The V2 coordinator follows the same architecture as review-coordinator.agent.md. 10 workflow steps map to the full spec lifecycle. Skill dispatch uses the verbatim Section 8.2 template with all 8 inputs.

**Code quality**: 365 lines (within 300-500 target). No em dashes, smart quotes, or curly apostrophes. No hardcoded values that belong in config. Encoding verified clean.

**Scope discipline**: Only `.github/agents/spec-architect.agent.md` was modified. No unrelated changes. Coordinator delegates all section writing to skills as designed.

**Coverage**: N/A - this is a markdown agent file, not executable code. Acceptance criteria serve as the verification mechanism.

**Documentation**: No .sdd/docs/ files affected (agent instructions are not documented in .sdd/docs/).

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T13:30:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (PASS), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (PASS), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 59 acceptance criteria checked off. All 22 FRs + FR-028 verified compliant.
- [PASS] Activity Log: Consistent entries showing planned -> doing -> for_review transitions.
- [WARN] Commit granularity: Single commit (627a2be) covers all 11 tasks. Acceptable for single-file rewrite but not ideal for task traceability.
- [PASS] Encoding: No prohibited Unicode characters found.

### Review Feedback

> No FAIL findings. No FB-XX items to address.

(none)

### Warnings
- [WARN] Single commit for 11 tasks (PROC-003). The entire V1-to-V2 rewrite of spec-architect.agent.md was committed as one change. Since this is a single-file markdown rewrite where tasks are not independently meaningful as separate commits, this is acceptable.

### Cross-Correlation Notes
No cross-correlation findings. All 8 skills produced consistent results with no conflicts, duplicates, or systemic patterns.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 23 | 0 | 0 |
| review-security | 5 | 0 | 0 |
| review-quality | 8 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 6 | 0 | 0 |
| review-performance | 1 | 0 | 0 |
| review-docs | 2 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **48** | **1** | **0** |

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T12:30:00Z - coder - lane=for_review - All 11 tasks complete, spec-architect.agent.md rewritten from V1 (530 lines) to V2 coordinator (365 lines), submitted for review
- 2026-04-05T13:30:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARN)
