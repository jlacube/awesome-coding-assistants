---
lane: doing
---

# WP34 - Code-Adjacent Doc Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/007-docs-agent.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP30, WP31 |
| Goal | Implement doc-changelog and doc-inline-code skills for code-adjacent documentation |
| Status | Not Started |
| Independent Test | Invoke each skill with a sample WP and verify CHANGELOG.md is prepended and source file docstrings are added |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP34-code-adjacent-doc-skills.md` |

## Objective

Implement the doc-changelog and doc-inline-code skills that produce code-adjacent documentation. doc-changelog appends structured changelog entries for each approved WP. doc-inline-code adds/updates docstrings, comments, and type annotations in implementation source files. These skills are unique: doc-changelog writes to a structured log format, and doc-inline-code modifies source code files (the only doc skill that does so).

## Spec References

FR-016, FR-017, FR-018, FR-019, FR-020, US-01, Section 4.2.5, Section 4.2.6, Section 7.1

## Tasks

### T34-01 - Create doc-changelog SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-changelog/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-016, Section 4.2.5
- **Parallel**: No
- **Acceptance criteria**:
  - [x] doc-changelog SKILL.md has valid YAML frontmatter with `name: doc-changelog`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies `.sdd/docs/CHANGELOG.md` as the target file
  - [x] Execution sequence defines the step-by-step process for generating changelog entries
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure
  - Pattern: Follow DOC-SKILL-CONTRACT.md
  - Files to modify: `.github/skills/doc-changelog/SKILL.md`

### T34-02 - Write changelog entry generation logic

- **Description**: Write the skill instructions for generating changelog entries. The skill SHALL append to `.sdd/docs/CHANGELOG.md` with: WP identifier and title, date, list of changes (from WP task list and descriptions), breaking changes (if any), and dependencies added/changed.
- **Spec refs**: FR-016
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL append to CHANGELOG.md with WP identifier and title (FR-016.1)
  - [x] The skill SHALL include the date of the changelog entry (FR-016.2)
  - [x] The skill SHALL include a list of changes derived from the WP task list and descriptions (FR-016.3)
  - [x] The skill SHALL include breaking changes if any (FR-016.4)
  - [x] The skill SHALL include dependencies added or changed (FR-016.5)
  - [x] BDD: Given WP03 is approved, When the Docs Agent runs, Then CHANGELOG.md has a new entry for WP03 (Section 11.2 Scenario 1)
- **Test requirements**: BDD
- **Depends on**: T34-01
- **Implementation Guidance**:
  - Pattern: Read WP task list to generate change descriptions; inspect WP implementation notes for breaking changes
  - Known pitfalls: Changes should be user-meaningful, not just task titles -- transform technical task descriptions into readable changelog entries
  - Files to modify: `.github/skills/doc-changelog/SKILL.md`

### T34-03 - Write changelog prepend ordering

- **Description**: Write the skill instructions enforcing that changelog entries are prepended (newest first), not appended to the end of the file.
- **Spec refs**: FR-017
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Changelog entries SHALL be prepended (newest first), not appended (FR-017)
  - [x] The most recent WP entry appears first in CHANGELOG.md after the file header
  - [x] Existing entries are preserved in their original order below the new entry
- **Test requirements**: BDD
- **Depends on**: T34-02
- **Implementation Guidance**:
  - Pattern: Read existing CHANGELOG.md, identify the insertion point (after the file header, before the first existing entry), insert the new entry
  - Known pitfalls: Do not simply append to the file -- FR-017 explicitly requires prepend ordering (newest first)
  - Files to modify: `.github/skills/doc-changelog/SKILL.md`

### T34-04 - Create doc-inline-code SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-inline-code/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-018, Section 4.2.6
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] doc-inline-code SKILL.md has valid YAML frontmatter with `name: doc-inline-code`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies "implementation source files" as the target (not `.sdd/docs/`)
  - [x] Execution sequence defines the step-by-step process for adding docstrings and comments
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure
  - Pattern: This is the only doc skill that modifies source files instead of `.sdd/docs/` files
  - Files to modify: `.github/skills/doc-inline-code/SKILL.md`

### T34-05 - Write docstring and comment generation logic

- **Description**: Write the skill instructions for adding/updating docstrings and comments in implementation source files. The skill SHALL add/update: module-level docstrings describing purpose, function/method docstrings with parameter descriptions, return types, and raises/throws, complex logic comments explaining "why" not "what", and type annotations if missing and the language supports them.
- **Spec refs**: FR-018
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL add/update module-level docstrings describing purpose (FR-018.1)
  - [x] The skill SHALL add/update function/method docstrings with parameter descriptions, return types, and raises/throws (FR-018.2)
  - [x] The skill SHALL add complex logic comments explaining "why", not "what" (FR-018.3)
  - [x] The skill SHALL add type annotations if missing and the language supports them (FR-018.4)
  - [x] The skill reads existing source files to determine which docstrings are missing or stale
- **Test requirements**: BDD
- **Depends on**: T34-04
- **Implementation Guidance**:
  - Pattern: Read source files modified by the WP (from git diff); for each file, analyze functions/methods/classes and add missing docstrings
  - Reference: Spec edge case: "source files have existing docstrings? doc-inline-code updates them if they are stale, leaves them if accurate"
  - Known pitfalls: Do not add redundant comments that repeat what code already says clearly; focus on "why" comments for complex logic
  - Files to modify: `.github/skills/doc-inline-code/SKILL.md`

### T34-06 - Write no-logic-modification constraint

- **Description**: Write the skill instructions explicitly prohibiting modification of implementation logic. The skill SHALL NOT modify implementation logic; it SHALL only add documentary content (docstrings, comments, type annotations).
- **Spec refs**: FR-019
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL NOT modify implementation logic; only add documentary content (FR-019)
  - [x] The constraint is prominently placed in the skill instructions (not buried in a footnote)
  - [x] Examples of prohibited modifications are listed (e.g., changing variable names, refactoring functions, fixing bugs)
- **Test requirements**: BDD
- **Depends on**: T34-05
- **Implementation Guidance**:
  - Pattern: Add a dedicated "Constraints" section to the SKILL.md with explicit prohibitions
  - Known pitfalls: LLM-based skills may be tempted to "improve" code while adding docs -- the constraint must be strong and clear
  - Files to modify: `.github/skills/doc-inline-code/SKILL.md`

### T34-07 - Write convention detection and adherence

- **Description**: Write the skill instructions for detecting and following the project's existing docstring convention. If a convention exists (e.g., NumPy style, Google style), the skill SHALL follow it. If none exists, the skill SHALL use the language's standard: Python = Google style, TypeScript = JSDoc, Go = godoc, Rust = rustdoc.
- **Spec refs**: FR-020
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL follow the project's existing docstring convention if one exists (FR-020)
  - [x] If no convention exists, the skill SHALL use the language standard: Python = Google style, TypeScript = JSDoc, Go = godoc, Rust = rustdoc (FR-020)
  - [x] The skill detects existing conventions by examining existing docstrings in the codebase
- **Test requirements**: BDD
- **Depends on**: T34-05
- **Implementation Guidance**:
  - Pattern: Read a sample of existing source files to detect docstring style; if consistent style found, follow it; if no docstrings exist, use language default
  - Reference: Write the Docs guide on docstring conventions for each language
  - Files to modify: `.github/skills/doc-inline-code/SKILL.md`

### T34-08 - Integration verification with coordinator

- **Description**: Verify both skills integrate correctly with the Docs Agent coordinator. Confirm the coordinator discovers doc-changelog and doc-inline-code, dispatches them in canonical order (positions 5 and 6), and passes the correct context. Verify that doc-inline-code's source file modifications are included in the coordinator's commit.
- **Spec refs**: FR-003, FR-004, FR-009
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator discovers both skills via `doc-*/SKILL.md` glob
  - [x] doc-changelog is dispatched at canonical position 5 (after doc-developer-guide)
  - [x] doc-inline-code is dispatched at canonical position 6 (last)
  - [x] Both skills receive all 6 context items defined in FR-005
  - [x] Source files modified by doc-inline-code are included in the coordinator's commit alongside `.sdd/docs/` changes (FR-009)
- **Test requirements**: BDD
- **Depends on**: T34-01, T34-02, T34-03, T34-04, T34-05, T34-06, T34-07
- **Implementation Guidance**:
  - Pattern: Manual invocation of the Docs Agent coordinator with a sample approved WP that has source files
  - Known pitfalls: The commit command must include both `.sdd/docs/` AND modified source files -- verify the coordinator's commit policy handles both

## Implementation Notes

doc-changelog follows a structured log format (newest-first prepend) rather than the incremental section-update pattern used by other doc skills. The changelog must preserve a consistent format across entries.

doc-inline-code is the only doc skill that modifies source files rather than `.sdd/docs/` files. This has implications for the coordinator's commit policy (FR-009): modified source files must be included alongside doc file changes. The skill must detect the project's docstring convention automatically.

All implementation artifacts are markdown SKILL.md files. "Testing" means manually invoking each skill with a sample WP and verifying the output.

## Risks & Mitigations

- **Risk**: doc-inline-code may accidentally modify implementation logic while adding docstrings. **Mitigation**: FR-019 constraint is prominently enforced in the skill instructions with explicit prohibitions.
- **Risk**: Changelog entries may become repetitive or overly technical. **Mitigation**: The skill should transform task descriptions into user-meaningful entries.
- **Risk**: Different programming languages have very different docstring conventions. **Mitigation**: FR-020 specifies explicit defaults per language (Google style for Python, JSDoc for TypeScript, etc.).

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T01:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T01:10:00Z - coder - T34-01/T34-02/T34-03 - completed - doc-changelog SKILL.md implemented with prepend ordering
- 2026-04-06T01:20:00Z - coder - T34-04/T34-05/T34-06/T34-07 - completed - doc-inline-code SKILL.md implemented with convention detection and no-logic constraint
- 2026-04-06T01:30:00Z - coder - T34-08 - completed - Integration verification passed: both skills discovered, correct canonical positions, all 6 context items, source files in commit
