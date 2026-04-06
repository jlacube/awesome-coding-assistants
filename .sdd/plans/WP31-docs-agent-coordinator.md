---
lane: done
---

# WP31 - Docs Agent Coordinator

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/007-docs-agent.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP30 |
| Goal | Create the Docs Agent coordinator that orchestrates 6 doc skills after WP approval |
| Status | Not Started |
| Independent Test | Invoke the Docs Agent with an approved WP and verify it discovers and dispatches all 6 doc skills in canonical order |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP31-docs-agent-coordinator.md` |

## Objective

Write the full Docs Agent coordinator in `.github/agents/docs-agent.agent.md`. The coordinator reads an approved WP's context, discovers doc skills dynamically, dispatches them sequentially in canonical order, handles skill failures gracefully, and commits documentation changes. This is the central orchestration logic that connects WP approval to documentation generation.

## Spec References

FR-001, FR-002, FR-003, FR-004, FR-005, FR-006, FR-007, FR-008, FR-009, Section 6.1 (Post-Approval Documentation Flow), Section 8.1 (Coordinator Invocation)

## Tasks

### T31-01 - Write YAML frontmatter for docs-agent.agent.md

- **Description**: Replace the placeholder content in `.github/agents/docs-agent.agent.md` with complete YAML frontmatter including name, description, model, tools list, handoffs, and argument-hint. The tools list SHALL include runSubagent, readFile, editFiles, createFile, createDirectory, fileSearch, textSearch, codebase, listDirectory, changes, fetch, runInTerminal, getTerminalOutput, and todo.
- **Spec refs**: FR-001, Section 8.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] YAML frontmatter includes `name`, `description`, `model`, `tools`, `handoffs`, and `argument-hint` fields
  - [x] `description` mentions triggering after WP approval and documentation generation
  - [x] `tools` list includes `agent/runSubagent` for skill dispatch
  - [x] `handoffs` includes a handoff back to the Coder or Review Coordinator if needed
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/agents/review-coordinator.agent.md` for YAML frontmatter format and tool naming conventions
  - Pattern: Use the same numbered naming convention (e.g., "6. Docs Agent")
  - Known pitfalls: Tool names must match VS Code Copilot Chat's internal tool registry exactly

### T31-02 - Write trigger context and artifact chain loading

- **Description**: Write the coordinator section that reads the approved WP context. The coordinator SHALL receive the WP file path, spec path, contracts directory, and implementation source files. It SHALL read the WP file and task list, the spec referenced by the WP, contract files in `.sdd/plans/contracts/<WP-slug>/`, implementation source files (from git diff), and existing documentation in `.sdd/docs/`.
- **Spec refs**: FR-001, FR-002
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL be triggered after a WP is reviewed and approved (lane = done) with the WP file path, spec path, contracts directory, and implementation source files (FR-001)
  - [x] The coordinator SHALL read the approved WP file and its task list, the spec, contract files, implementation source files from git diff, and existing documentation in `.sdd/docs/` (FR-002)
  - [x] If no WP path is provided, the coordinator SHALL halt with a descriptive error message (FR-001 error)
  - [x] If a referenced file does not exist, the coordinator SHALL log a warning and proceed with available files; if the WP file itself is missing, the coordinator SHALL halt (FR-002 error)
- **Test requirements**: BDD
- **Depends on**: T31-01
- **Implementation Guidance**:
  - Reference: Section 6.1 steps 1-3 for the complete loading sequence
  - Reference: Section 8.1 for the prompt template that triggers the coordinator
  - Pattern: Use `read_file` or `readFile` tool to load each artifact; use `get_changed_files` or `git diff` to identify implementation source files
  - Error handling: Two-tier error model -- halt if WP is missing (critical), warn-and-continue if other files are missing (best-effort)
  - Files to modify: `.github/agents/docs-agent.agent.md`

### T31-03 - Write dynamic skill discovery

- **Description**: Write the coordinator section that discovers doc skills by scanning `.github/skills/doc-*/SKILL.md`. If zero skills are found, the coordinator SHALL halt and report.
- **Spec refs**: FR-003
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL discover doc skills by scanning `.github/skills/doc-*/SKILL.md` (FR-003)
  - [x] If zero skills are found, the coordinator SHALL halt and report "No doc skills found" (FR-003 error)
  - [x] Discovery results are logged showing which skills were found
- **Test requirements**: BDD
- **Depends on**: T31-02
- **Implementation Guidance**:
  - Reference: Review Coordinator's dynamic discovery of `review-*/SKILL.md` for the established pattern
  - Pattern: Use `file_search` with glob pattern `doc-*/SKILL.md`
  - Known pitfalls: The glob must match from `.github/skills/` -- verify the base path

### T31-04 - Write canonical ordering and sequential dispatch

- **Description**: Write the coordinator section that sorts discovered skills into canonical order and dispatches each sequentially as a subagent. The canonical order is: doc-architecture, doc-api-reference, doc-user-guide, doc-developer-guide, doc-changelog, doc-inline-code. Each skill receives the full context specified in FR-005.
- **Spec refs**: FR-004, FR-005, FR-006
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The coordinator SHALL dispatch skills in canonical order: doc-architecture, doc-api-reference, doc-user-guide, doc-developer-guide, doc-changelog, doc-inline-code (FR-004)
  - [x] Each skill SHALL be dispatched as a subagent with: skill file path, WP file path and task list, spec path and contract files, implementation source files, existing docs directory, active doc-domain patterns (FR-005)
  - [x] Skills SHALL execute sequentially, each reading existing docs before writing updates (FR-006)
  - [x] Skills that are discovered but not in the canonical list SHALL be dispatched after all canonical skills, in alphabetical order
- **Test requirements**: BDD
- **Depends on**: T31-03
- **Implementation Guidance**:
  - Reference: Planner coordinator's two-phase dispatch for sequential subagent pattern
  - Pattern: Build a prompt template for each skill dispatch that includes all 6 context items from FR-005
  - Known pitfalls: Skills update docs incrementally -- each subsequent skill must read the current state of `.sdd/docs/` to avoid overwriting prior skill's changes (FR-006)
  - Files to modify: `.github/agents/docs-agent.agent.md`

### T31-05 - Write skill failure tolerance

- **Description**: Write the coordinator section that handles individual skill failures. If any skill fails, the coordinator SHALL log the failure (skill name, error description) and continue to the next skill. Documentation generation is best-effort.
- **Spec refs**: FR-007
- **Parallel**: No
- **Acceptance criteria**:
  - [x] If a skill fails, the coordinator SHALL log the failure and continue to the next skill (FR-007)
  - [x] The failure log includes the skill name and a description of the error
  - [x] A failed skill does not prevent subsequent skills from executing
  - [x] BDD: Given doc-changelog encounters an error, When the coordinator detects the failure, Then it logs the error And continues to doc-inline-code (Section 11.2 Scenario 3)
- **Test requirements**: BDD
- **Depends on**: T31-04
- **Implementation Guidance**:
  - Reference: Coder coordinator's debug retry logic for error handling patterns
  - Pattern: Wrap each `runSubagent` call in error-handling instructions; if subagent fails, log and proceed
  - Known pitfalls: Do not halt on any individual skill failure -- the coordinator must be resilient

### T31-06 - Write doc-patterns consumption

- **Description**: Write the coordinator section that reads `.sdd/reviews/doc-patterns.md` before dispatching skills. If the file does not exist, proceed without patterns.
- **Spec refs**: FR-008
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] The coordinator SHALL read `.sdd/reviews/doc-patterns.md` before dispatching skills (FR-008)
  - [x] If doc-patterns.md does not exist, the coordinator proceeds without patterns (no error)
  - [x] Active patterns are passed to each skill as part of the dispatch context (FR-005)
- **Test requirements**: none
- **Depends on**: T31-01
- **Implementation Guidance**:
  - Reference: Planner coordinator's patterns consumption section (reads plan-patterns.md)
  - Pattern: Use `read_file` with a conditional check; if file not found, set patterns to empty
  - Files to modify: `.github/agents/docs-agent.agent.md`

### T31-07 - Write commit policy

- **Description**: Write the coordinator section that commits documentation changes after all skills complete. The commit SHALL include `.sdd/docs/` and any modified source files (from doc-inline-code). If no documentation files were modified, skip the commit and log that no updates were needed.
- **Spec refs**: FR-009
- **Parallel**: No
- **Acceptance criteria**:
  - [x] After all skills complete, the coordinator SHALL commit documentation changes with `git add .sdd/docs/ <modified source files>` followed by `git commit -m "docs(docs): update documentation for WP<NN>"` (FR-009)
  - [x] If no documentation files were modified (all skills produced no output), the coordinator SHALL skip the commit and log that no updates were needed (FR-009 error)
  - [x] If the git commit fails, the coordinator SHALL report the error to the invoker (FR-009 error)
  - [x] Source files modified by doc-inline-code are included in the commit
- **Test requirements**: BDD
- **Depends on**: T31-04
- **Implementation Guidance**:
  - Reference: Spec 007 Section 4.1.5 for exact commit message format
  - Pattern: Use `run_in_terminal` for git commands; list files explicitly (do not use `git add .`)
  - Error handling: Three cases -- successful commit, no changes to commit, git error
  - Known pitfalls: Source files modified by doc-inline-code must be explicitly added -- do not rely on `.sdd/docs/` alone

### T31-08 - Verify encoding compliance

- **Description**: Verify the completed docs-agent.agent.md uses UTF-8 encoding, LF line endings, plain ASCII hyphens (no em dashes), straight quotes (no smart quotes), and no curly apostrophes.
- **Spec refs**: Section 9.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] docs-agent.agent.md is UTF-8 encoded with no BOM
  - [x] All line endings are LF (not CRLF)
  - [x] No em dashes, smart quotes, or curly apostrophes appear in the file
- **Test requirements**: none
- **Depends on**: T31-01, T31-02, T31-03, T31-04, T31-05, T31-06, T31-07
- **Implementation Guidance**:
  - Pattern: Run encoding verification as a final gating task
  - Known pitfalls: Windows environments may introduce CRLF

## Implementation Notes

The Docs Agent coordinator follows the same structural pattern as the Review Coordinator (`.github/agents/review-coordinator.agent.md`) and Coder coordinator (`.github/agents/coder.agent.md`): YAML frontmatter declaring tools and handoffs, followed by procedural instructions for context loading, skill discovery, dispatch, and commit.

The key architectural decision (Section 9.2 Decision 1) is that the Docs Agent is a dedicated coordinator separate from the Coder. The Coder no longer maintains `.sdd/docs/`. The Docs Agent runs after every WP approval (Section 9.2 Decision 2), producing incremental documentation updates rather than bulk updates at the end.

All implementation artifacts are markdown files. "Testing" means manually invoking the coordinator with an approved WP and verifying that doc skills are discovered, dispatched, and documentation files are updated.

## Risks & Mitigations

- **Risk**: The Orchestrator may not know when to trigger the Docs Agent. **Mitigation**: The Docs Agent is triggered explicitly by the Review Coordinator or Orchestrator after setting lane = done. Spec 008 (Orchestrator V2) will define the integration point.
- **Risk**: Sequential dispatch of 6 skills may approach the 20-minute NFR-001 limit. **Mitigation**: Each skill focuses on a single doc dimension, keeping individual execution fast.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-06T13:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-quality (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked (T31-01 through T31-08)
- [PASS] Activity Log: Correct lane transitions (planned -> doing -> for_review)
- [WARN] Commit granularity: Single commit for all 8 tasks (706ab3d)
- [PASS] Encoding: No violations found (UTF-8, LF, no prohibited Unicode)

### Review Feedback

No FAIL findings. No remediation required.

### Warnings
- [WARN] Commit granularity: All 8 tasks (T31-01 through T31-08) committed in a single commit (706ab3d). Since all tasks produce content within a single file (docs-agent.agent.md), this is a practical limitation rather than a process failure. (PROC-003)

### Cross-Correlation Notes
No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 9 | 0 | 0 |
| review-quality | 6 | 0 | 0 |
| **Total** | **18** | **1** | **0** |

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:15:00Z - coder - lane=for_review - All tasks complete, encoding verified
- 2026-04-06T13:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARN)
