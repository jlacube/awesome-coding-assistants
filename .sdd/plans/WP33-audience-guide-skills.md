---
lane: doing
---

# WP33 - Audience Guide Doc Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/007-docs-agent.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP30, WP31 |
| Goal | Implement doc-user-guide and doc-developer-guide skills for audience-facing documentation |
| Status | Not Started |
| Independent Test | Invoke each skill with a sample WP and verify user-guide.md and developer-guide.md are updated correctly |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP33-audience-guide-skills.md` |

## Objective

Implement the doc-user-guide and doc-developer-guide skills that produce audience-facing documentation. doc-user-guide generates/updates end-user documentation describing features, workflows, and troubleshooting. doc-developer-guide generates/updates developer onboarding documentation covering setup, conventions, and contribution patterns. These skills target different audiences (end users vs developers) but share the same structural pattern.

## Spec References

FR-014, FR-015, US-01, Section 4.2.3, Section 4.2.4, Section 7.1

## Tasks

### T33-01 - Create doc-user-guide SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-user-guide/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-014, Section 4.2.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] doc-user-guide SKILL.md has valid YAML frontmatter with `name: doc-user-guide`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies `.sdd/docs/user-guide.md` as the target file
  - [x] Execution sequence defines the step-by-step process for generating user guide docs
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure
  - Pattern: Follow DOC-SKILL-CONTRACT.md
  - Files to modify: `.github/skills/doc-user-guide/SKILL.md`

### T33-02 - Write user guide generation sections

- **Description**: Write the skill instructions for generating user guide documentation. The skill SHALL produce/update `.sdd/docs/user-guide.md` with: feature descriptions (from user stories), step-by-step usage instructions, configuration options (from config schema contract), common workflows, and troubleshooting for expected error scenarios.
- **Spec refs**: FR-014
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL produce/update `.sdd/docs/user-guide.md` with feature descriptions from user stories (FR-014.1)
  - [x] The skill SHALL include step-by-step usage instructions (FR-014.2)
  - [x] The skill SHALL include configuration options from config schema contract (FR-014.3)
  - [x] The skill SHALL include common workflows (FR-014.4)
  - [x] The skill SHALL include troubleshooting for expected error scenarios (FR-014.5)
  - [x] On incremental updates, the skill SHALL update affected sections without overwriting unrelated sections (inherits FR-011 pattern)
- **Test requirements**: BDD
- **Depends on**: T33-01
- **Implementation Guidance**:
  - Pattern: Source feature descriptions from user stories in the spec; source config options from config-schema contract files if they exist
  - Known pitfalls: User stories may not exist for all features -- the skill should also derive feature descriptions from FR descriptions
  - Reference: Edge case from spec: "What happens when a WP changes no API endpoints? doc-api-reference has no updates." Same principle applies here -- if a WP adds no user-facing features, the skill exits cleanly
  - Files to modify: `.github/skills/doc-user-guide/SKILL.md`

### T33-03 - Create doc-developer-guide SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-developer-guide/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-015, Section 4.2.4
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] doc-developer-guide SKILL.md has valid YAML frontmatter with `name: doc-developer-guide`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies `.sdd/docs/developer-guide.md` as the target file
  - [x] Execution sequence defines the step-by-step process for generating developer guide docs
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure
  - Pattern: Follow DOC-SKILL-CONTRACT.md
  - Files to modify: `.github/skills/doc-developer-guide/SKILL.md`

### T33-04 - Write developer guide generation sections

- **Description**: Write the skill instructions for generating developer guide documentation. The skill SHALL produce/update `.sdd/docs/developer-guide.md` with: development environment setup, project structure overview, coding conventions in use, testing approach and commands, and how to add new features following the project's patterns.
- **Spec refs**: FR-015
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL produce/update `.sdd/docs/developer-guide.md` with development environment setup (FR-015.1)
  - [x] The skill SHALL include project structure overview (FR-015.2)
  - [x] The skill SHALL include coding conventions in use (FR-015.3)
  - [x] The skill SHALL include testing approach and commands (FR-015.4)
  - [x] The skill SHALL include how to add new features following the project's patterns (FR-015.5)
  - [x] On incremental updates, the skill SHALL update affected sections without overwriting unrelated sections (inherits FR-011 pattern)
- **Test requirements**: BDD
- **Depends on**: T33-03
- **Implementation Guidance**:
  - Pattern: Source project structure from actual codebase (use `list_dir`, `file_search`); source conventions from existing code patterns and style files
  - Known pitfalls: Development setup instructions should be derived from actual project files (package.json, requirements.txt, go.mod, Cargo.toml), not invented
  - Reference: BDD Scenario 1 from Section 11.2: "developer-guide.md is updated" after docs agent runs
  - Files to modify: `.github/skills/doc-developer-guide/SKILL.md`

### T33-05 - Integration verification with coordinator

- **Description**: Verify both skills integrate correctly with the Docs Agent coordinator. Confirm the coordinator discovers doc-user-guide and doc-developer-guide, dispatches them in canonical order (positions 3 and 4), and passes the correct context.
- **Spec refs**: FR-003, FR-004
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Coordinator discovers both skills via `doc-*/SKILL.md` glob
  - [ ] doc-user-guide is dispatched at canonical position 3 (after doc-api-reference)
  - [ ] doc-developer-guide is dispatched at canonical position 4 (after doc-user-guide)
  - [ ] Both skills receive all 6 context items defined in FR-005
- **Test requirements**: BDD
- **Depends on**: T33-01, T33-02, T33-03, T33-04
- **Implementation Guidance**:
  - Pattern: Manual invocation of the Docs Agent coordinator with a sample approved WP
  - Known pitfalls: Verify that user-guide.md and developer-guide.md existing content is preserved during incremental updates

## Implementation Notes

Both skills produce markdown content for `.sdd/docs/`. They follow identical structural patterns but target different audiences:
- doc-user-guide targets end users who consume the product's features
- doc-developer-guide targets developers who contribute to the codebase

Both skills inherit the incremental update pattern from FR-011: read existing content before writing, update only affected sections.

All implementation artifacts are markdown SKILL.md files. "Testing" means manually invoking each skill with a sample WP and verifying the output docs.

## Risks & Mitigations

- **Risk**: User guide content may overlap with developer guide content (e.g., configuration). **Mitigation**: User guide focuses on "how to use" while developer guide focuses on "how to build/contribute". The skill instructions should clearly delineate scope.
- **Risk**: First-run (no existing docs) vs incremental update behavior differs. **Mitigation**: Spec edge case states "Skills create the doc files from scratch" on first WP. Skills should handle both cases.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:01:00Z - coder - T33-01 - completed - doc-user-guide SKILL.md structure created with frontmatter, input/output contracts, execution sequence
- 2026-04-06T12:02:00Z - coder - T33-02 - completed - User guide generation sections written: features, usage instructions, configuration, workflows, troubleshooting, incremental update protocol
- 2026-04-06T12:03:00Z - coder - T33-03 - completed - doc-developer-guide SKILL.md structure created with frontmatter, input/output contracts, execution sequence
- 2026-04-06T12:04:00Z - coder - T33-04 - completed - Developer guide generation sections written: env setup, project structure, conventions, testing, adding features, incremental update protocol
