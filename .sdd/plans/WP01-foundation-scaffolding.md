---
lane: planned
---

# WP01 - Foundation & Scaffolding

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P0 |
| Lane | planned |
| Depends on | none |
| Goal | Establish directory structure, deprecate old reviewer, create initial review artifacts, and update Orchestrator reference so subsequent WPs can build on a clean foundation |
| Status | Not Started |
| Independent Test | Verify: `.sdd/reviews/` directory exists, `review-patterns.md` contains the initial template from Section 7.3, `reviewer.agent.md` is renamed to `.deprecated`, skill directories for P1 skills exist under `.github/skills/`, and Orchestrator references "5. Review Coordinator" |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP01-foundation-scaffolding.md` |

## Objective

Set up the directory structure, artifact templates, and agent references required by the Reviewer V2 skill-based architecture. This WP delivers no review logic - it creates the scaffolding that WP02-WP07 build upon. It also deprecates the old monolithic reviewer agent and makes the minimal Orchestrator reference change needed for pipeline continuity.

## Spec References

- Section 7.3 (Review Patterns File format)
- Section 9.3 (Directory & Module Structure)
- Section 9.4 Decision 1 (Dynamic skill discovery)
- Section 7.4 (Coordinator Agent File - name field)
- Constraint C-006 (Existing SDD pipeline contract preserved)

## Tasks

### T01-01 - Create review artifacts directory structure

- **Description**: Create the `.sdd/reviews/` directory that will hold per-WP review findings and the patterns file. This is the root directory referenced by FR-008, FR-010, FR-014, and FR-018.
- **Spec refs**: Section 9.3 Directory Structure
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] Directory `.sdd/reviews/` exists in the workspace root
  - [ ] Directory is committed to version control (not gitignored)
  - [ ] A `.gitkeep` file is placed in `.sdd/reviews/` to ensure the empty directory is tracked by Git
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Create the directory using `mkdir -p .sdd/reviews/`
  - Add a `.gitkeep` file so Git tracks the empty directory
  - Do NOT create per-WP subdirectories (e.g., `WP01-*/`) - those are created at review time by the coordinator per FR-008

### T01-02 - Create P1 review skill directory structure

- **Description**: Create the three P1 review skill directories under `.github/skills/`. Each directory will later receive its `SKILL.md` file in WP03-WP05. The directory must exist for the coordinator's dynamic discovery glob (FR-003) to work.
- **Spec refs**: Section 9.3 Directory Structure, FR-003 (glob pattern `.github/skills/review-*/SKILL.md`)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] Directory `.github/skills/review-spec/` exists
  - [ ] Directory `.github/skills/review-security/` exists
  - [ ] Directory `.github/skills/review-quality/` exists
  - [ ] Each directory contains a `.gitkeep` file for Git tracking
  - [ ] Directory names exactly match the canonical skill names from FR-004 dispatch order
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Create all three directories: `review-spec`, `review-security`, `review-quality`
  - Do NOT create P2/P3 skill directories yet - those will be created in WP06/WP07
  - The directory names MUST match the glob pattern `.github/skills/review-*/` used by FR-003
  - Reference existing skill directory structure: `.github/skills/semantic-commit/SKILL.md` as the pattern

### T01-03 - Create initial review-patterns.md template

- **Description**: Create the review patterns file at `.sdd/reviews/review-patterns.md` using the exact format from Section 7.3. This file will be curated by the coordinator after each review (FR-018). It must exist with the correct structure before the first review.
- **Spec refs**: Section 7.3 (Review Patterns File), FR-018 (patterns curation), FR-019 (no patterns from WARNs)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] File exists at `.sdd/reviews/review-patterns.md`
  - [ ] File contains the header with `Last updated` and `Last review` fields (initially empty/placeholder)
  - [ ] File contains the Coder instruction paragraph: "Coder: read this file before implementing any WP..."
  - [ ] File contains `## Active Patterns` section (initially empty)
  - [ ] File contains `## Resolved` section (initially empty)
  - [ ] File uses only plain ASCII hyphens and straight quotes (no em dashes, smart quotes)
- **Test requirements**: none (format verification)
- **Depends on**: T01-01 (reviews directory must exist)
- **Implementation Guidance**:
  - Use the exact template from spec Section 7.3, but with placeholder values:
    ```markdown
    # Review Patterns

    > Last updated: (none)
    > Last review: (none)

    Coder: read this file before implementing any WP. These patterns document
    mistakes caught in previous reviews. Avoid repeating them.

    ## Active Patterns

    (No patterns recorded yet.)

    ## Resolved

    (No resolved patterns yet.)
    ```
  - The coordinator will populate this file after the first review (FR-018)
  - Pattern IDs follow `PAT-NNN` format (three-digit zero-padded)
  - Only FAIL findings generate patterns (FR-019) - document this in the file header if helpful

### T01-04 - Deprecate old reviewer.agent.md

- **Description**: Rename the existing monolithic reviewer agent file from `reviewer.agent.md` to `reviewer.agent.md.deprecated` per Section 9.3. The old file is preserved for reference but must not be active.
- **Spec refs**: Section 9.3 ("DEPRECATED: kept for reference, renamed to reviewer.agent.md.deprecated")
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] File `.github/agents/reviewer.agent.md` no longer exists as an active agent file
  - [ ] File `.github/agents/reviewer.agent.md.deprecated` exists with the original content intact
  - [ ] The rename is performed via `git mv` to preserve history
  - [ ] No other agent files reference "5. Reviewer" by the old filename
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Use `git mv .github/agents/reviewer.agent.md .github/agents/reviewer.agent.md.deprecated`
  - This preserves Git history for the file
  - The `.deprecated` extension ensures VS Code does not load it as an active agent
  - Verify the Orchestrator update (T01-05) accounts for the name change

### T01-05 - Update Orchestrator agent reference

- **Description**: Update the Orchestrator agent's decision table to reference "5. Review Coordinator" instead of "5. Reviewer". This is a minimal name-string change required for pipeline continuity (C-006), NOT the larger "auto-continuation removal" that is out of scope (Section 13, A-004).
- **Spec refs**: Section 7.4 (coordinator name: "5. Review Coordinator"), C-006 (pipeline contract preserved), A-004 (larger Orchestrator changes are separate)
- **Parallel**: No (depends on understanding of T01-04)
- **Acceptance criteria**:
  - [ ] The Orchestrator agent file references "5. Review Coordinator" (or equivalent) where it previously referenced "5. Reviewer"
  - [ ] The Orchestrator's `lane: for_review` routing still delegates to the review agent
  - [ ] No other routing logic in the Orchestrator is modified (auto-continuation, next-WP scanning remain as-is)
  - [ ] The handoff label and description are updated to match the new agent name
- **Test requirements**: none (string replacement verification)
- **Depends on**: T01-04 (old reviewer must be deprecated first)
- **Implementation Guidance**:
  - Read `.github/agents/orchestrator.agent.md` and find all references to "5. Reviewer" or "Reviewer"
  - Replace with "5. Review Coordinator" or "Review Coordinator" as appropriate
  - ONLY change the name reference - do NOT modify routing logic, decision table conditions, or pipeline flow
  - The Orchestrator's auto-continuation behavior (scanning for next WPs after review) is intentionally left unchanged - that is a separate effort per A-004
  - Verify the Orchestrator's handoff entries in YAML frontmatter reference the correct agent name

### T01-06 - Verify directory structure and commit

- **Description**: Verify the complete directory structure matches Section 9.3, then commit all scaffolding changes as a single atomic commit.
- **Spec refs**: Section 9.3 (full directory tree)
- **Parallel**: No (final verification task)
- **Acceptance criteria**:
  - [ ] `.sdd/reviews/` exists with `.gitkeep`
  - [ ] `.sdd/reviews/review-patterns.md` exists with correct template
  - [ ] `.github/skills/review-spec/` exists with `.gitkeep`
  - [ ] `.github/skills/review-security/` exists with `.gitkeep`
  - [ ] `.github/skills/review-quality/` exists with `.gitkeep`
  - [ ] `.github/agents/reviewer.agent.md.deprecated` exists
  - [ ] `.github/agents/reviewer.agent.md` does NOT exist
  - [ ] Orchestrator references "5. Review Coordinator"
  - [ ] All files committed with explicit `git add` listing
- **Test requirements**: none (structural verification)
- **Depends on**: T01-01, T01-02, T01-03, T01-04, T01-05
- **Implementation Guidance**:
  - Run `ls -la` or equivalent on each directory to verify structure
  - Read the Orchestrator file to confirm the name reference change
  - Commit with: `git add .sdd/reviews/.gitkeep .sdd/reviews/review-patterns.md .github/skills/review-spec/.gitkeep .github/skills/review-security/.gitkeep .github/skills/review-quality/.gitkeep .github/agents/reviewer.agent.md.deprecated .github/agents/orchestrator.agent.md`
  - Commit message: `docs(WP01): scaffold reviewer v2 directory structure and deprecate old reviewer`
  - Use explicit file list in `git add` - never `git add .`

## Implementation Notes

- This WP creates NO review logic - it is purely structural scaffolding
- All files are markdown or empty placeholder files (.gitkeep)
- The old `reviewer.agent.md` is preserved as `.deprecated` for reference, not deleted
- The Orchestrator update is intentionally minimal (name string only) per the out-of-scope boundary in Section 13
- P2/P3 skill directories are NOT created in this WP - they are created in WP06/WP07 respectively

## Parallel Opportunities

- T01-01, T01-02, T01-03, T01-04 can all run concurrently (independent directory/file creation)
- T01-05 depends on T01-04 (must understand the deprecation before updating references)
- T01-06 depends on all prior tasks (final verification)

## Risks & Mitigations

- **Risk**: Orchestrator name change breaks pipeline routing
  - Mitigation: Only change the name string, verify routing logic is unchanged. Test by checking the Orchestrator's decision table still routes `lane: for_review` to the review agent.
- **Risk**: Deprecated reviewer file is still loaded by VS Code
  - Mitigation: The `.deprecated` extension should prevent VS Code from treating it as an agent. Verify by checking VS Code agent discovery behavior.

## Activity Log

- 2026-04-04T11:10:00Z - planner - lane=planned - Work package created
