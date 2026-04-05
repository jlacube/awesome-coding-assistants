---
lane: for_review
---

# WP30 - Foundation & Doc Skill Scaffolding

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/007-docs-agent.spec.md` |
| Priority | P0 |
| Lane | planned |
| Depends on | none |
| Goal | Establish directory structure and stub files enabling doc skill discovery |
| Status | Complete |
| Independent Test | Run `ls .github/skills/doc-*/SKILL.md` and verify 6 stub files exist |
| Parallelisable | - |
| Prompt | `.sdd/plans/WP30-foundation-doc-skills.md` |

## Objective

Create the directory scaffolding for all 6 documentation skills, stub SKILL.md files enabling dynamic discovery, a common doc skill contract, and a placeholder docs-agent coordinator file. This WP delivers no documentation logic but establishes the file structure that the coordinator and skills depend on.

## Spec References

FR-003 (dynamic skill discovery), FR-005 (skill dispatch parameters), Section 9.1 (directory structure), Section 9.2 (design decisions)

## Tasks

### T30-01 - Create doc skill directory structure

- **Description**: Create 6 directories under `.github/skills/`: `doc-architecture/`, `doc-api-reference/`, `doc-user-guide/`, `doc-developer-guide/`, `doc-changelog/`, `doc-inline-code/`.
- **Spec refs**: FR-003, Section 9.1
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] All 6 directories exist under `.github/skills/`
  - [x] Directory names match the canonical skill names from FR-004 exactly
  - [x] No extraneous directories are created
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: Section 9.1 lists the exact directory names
  - Pattern: Matches `.github/skills/review-*/` and `.github/skills/code-*/` naming convention used by Specs 001 and 004
  - Files to create: 6 empty directories (SKILL.md created in T30-02)

### T30-02 - Create stub SKILL.md files with YAML frontmatter

- **Description**: Create a stub SKILL.md file in each of the 6 doc skill directories. Each stub includes YAML frontmatter with `name`, `description`, and `argument-hint` fields. The body contains a placeholder noting the skill is pending implementation.
- **Spec refs**: FR-003, Section 9.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each of the 6 directories has a SKILL.md file
  - [x] Each SKILL.md has valid YAML frontmatter with `name` matching the directory name
  - [x] The glob pattern `doc-*/SKILL.md` returns all 6 files
- **Test requirements**: none
- **Depends on**: T30-01
- **Implementation Guidance**:
  - Reference: Existing `.github/skills/review-spec/SKILL.md` for YAML frontmatter format
  - Pattern: `argument-hint` should read "Invoked by Docs Agent Coordinator - do not call directly"
  - Files to create: `.github/skills/doc-architecture/SKILL.md`, `.github/skills/doc-api-reference/SKILL.md`, `.github/skills/doc-user-guide/SKILL.md`, `.github/skills/doc-developer-guide/SKILL.md`, `.github/skills/doc-changelog/SKILL.md`, `.github/skills/doc-inline-code/SKILL.md`

### T30-03 - Create DOC-SKILL-CONTRACT.md common contract

- **Description**: Create `.github/skills/DOC-SKILL-CONTRACT.md` defining the common input/output contract shared by all 6 doc skills. The contract specifies what context the coordinator provides (WP file, spec, contracts, source files, existing docs, patterns) and what each skill produces (updated doc files).
- **Spec refs**: FR-005, FR-006
- **Parallel**: No
- **Acceptance criteria**:
  - [x] DOC-SKILL-CONTRACT.md exists at `.github/skills/DOC-SKILL-CONTRACT.md`
  - [x] Input contract lists all 6 context items from FR-005: skill file path, WP file path and task list, spec path and contract files, implementation source files, existing docs directory, active doc-domain patterns
  - [x] Output contract defines what each skill SHALL produce: updated documentation files in `.sdd/docs/` or updated source files (for doc-inline-code)
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/PLAN-SKILL-CONTRACT.md` and `.github/skills/CODER-SKILL-CONTRACT.md` for format
  - Pattern: Follow the established contract structure: Input Contract table, Output Contract table, Error Handling section
  - Known pitfalls: The contract must include "existing docs directory" as an input since skills update incrementally (FR-006), not recreate from scratch

### T30-04 - Create missing doc output files

- **Description**: Ensure all documentation output files exist in `.sdd/docs/`. Currently `architecture.md`, `developer-guide.md`, and `user-guide.md` exist. Create `api-reference.md` and `CHANGELOG.md` with placeholder content.
- **Spec refs**: Section 7.1, Section 9.1
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] `.sdd/docs/api-reference.md` exists with a placeholder header
  - [x] `.sdd/docs/CHANGELOG.md` exists with a placeholder header
  - [x] Existing files (`architecture.md`, `developer-guide.md`, `user-guide.md`) are not modified
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Files to create: `.sdd/docs/api-reference.md`, `.sdd/docs/CHANGELOG.md`
  - Pattern: Each placeholder should have a top-level heading matching Section 7.1's content description
  - Known pitfalls: Do NOT overwrite existing doc files -- they may contain content from prior WPs

### T30-05 - Create docs-agent.agent.md placeholder

- **Description**: Create `.github/agents/docs-agent.agent.md` with minimal YAML frontmatter (name, description, model, tools) and a placeholder body. The full coordinator logic is written in WP31.
- **Spec refs**: Section 9.1, FR-001
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] `.github/agents/docs-agent.agent.md` exists
  - [x] YAML frontmatter includes `name`, `description`, `model`, and `tools` fields
  - [x] The agent is discoverable by the Orchestrator
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/agents/review-coordinator.agent.md` for YAML frontmatter format
  - Pattern: Name should be numbered to fit the pipeline sequence (e.g., "6. Docs Agent" or similar)
  - Known pitfalls: The Orchestrator must be able to find this agent -- verify the naming convention

### T30-06 - Verify directory structure and encoding compliance

- **Description**: Confirm all directories, stub files, contract, and placeholder files are in place. Verify UTF-8 encoding, LF line endings, no em dashes, no smart quotes, no curly apostrophes across all created files.
- **Spec refs**: Section 9.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] `ls .github/skills/doc-*/SKILL.md` returns exactly 6 files
  - [x] `.github/skills/DOC-SKILL-CONTRACT.md` exists
  - [x] `.github/agents/docs-agent.agent.md` exists
  - [x] `.sdd/docs/api-reference.md` and `.sdd/docs/CHANGELOG.md` exist
  - [x] All files are UTF-8 encoded with no BOM, LF line endings, no em dashes or smart quotes
- **Test requirements**: none
- **Depends on**: T30-01, T30-02, T30-03, T30-04, T30-05
- **Implementation Guidance**:
  - Pattern: Run encoding verification as a final gating task before committing
  - Known pitfalls: Windows environments may introduce CRLF -- verify LF

## Implementation Notes

All implementation artifacts are markdown files. There is no executable code, build system, or test framework. This is a pure scaffolding WP that creates the directory structure and file stubs required by the coordinator's dynamic discovery mechanism (FR-003).

The project does not use a language with package isolation (Python venv, Node.js node_modules, etc.), so no virtual environment setup task is included. All artifacts are plain markdown files consumed by the VS Code Copilot Chat agent framework.

## Risks & Mitigations

- **Risk**: doc-patterns.md may not exist if Spec 006 WP28 has not been implemented yet. **Mitigation**: FR-008 reads doc-patterns.md only if it exists. The coordinator handles its absence gracefully.
- **Risk**: Naming collision with existing skill directories. **Mitigation**: Verify no `doc-*` directories exist before creating. The `file_search` for `doc-*/SKILL.md` currently returns zero results.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:01:00Z - coder - T30-01 - completed - Created 6 doc skill directories
- 2026-04-06T12:01:00Z - coder - T30-02 - completed - Created 6 stub SKILL.md files with YAML frontmatter
- 2026-04-06T12:02:00Z - coder - T30-03 - completed - Created DOC-SKILL-CONTRACT.md
- 2026-04-06T12:02:00Z - coder - T30-04 - completed - Created api-reference.md and CHANGELOG.md placeholders
- 2026-04-06T12:02:00Z - coder - T30-05 - completed - Created docs-agent.agent.md placeholder
- 2026-04-06T12:03:00Z - coder - T30-06 - completed - Verified structure and encoding compliance
- 2026-04-06T12:04:00Z - coder - lane=for_review - All tasks complete, verification passed
