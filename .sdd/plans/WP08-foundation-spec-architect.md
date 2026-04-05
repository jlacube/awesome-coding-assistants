---
lane: done
---

# WP08 - Foundation & Skill Directories

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/002-spec-architect-v2.spec.md` |
| Priority | P0 |
| Lane | for_review |
| Depends on | none |
| Goal | Create the directory structure, common skill template, and infrastructure for Spec Architect V2 |
| Status | Complete |
| Independent Test | Verify: 8 directories exist at `.github/skills/spec-*/`; each contains a stub SKILL.md with valid YAML frontmatter; `.sdd/reviews/spec-patterns.md` exists |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP08-foundation-spec-architect.md` |

## Objective

Establish the directory layout, common skill template, and supporting infrastructure that all subsequent WPs depend on. This WP produces no functional logic -- it creates the scaffolding into which the coordinator and skills will be written. It follows the same pattern as WP01 (Reviewer V2 foundation), adapted for spec skills.

## Spec References

- FR-008 (artifacts directory creation)
- FR-009 (skill discovery via `.github/skills/spec-*/SKILL.md` glob)
- FR-010 (canonical skill ordering -- skill names must match)
- FR-014, FR-015, FR-016 (companion artifact file naming and target language)
- FR-019 (spec-patterns.md consumption)
- FR-023 through FR-028 (common skill contract)
- Section 9.3 (directory and module structure)

## Tasks

### T08-01 - Create spec skill directory structure

- **Description**: Create 8 directories under `.github/skills/` matching the glob pattern `spec-*/` used by the coordinator for dynamic discovery (FR-009). Directory names must exactly match the canonical order defined in FR-010.
- **Spec refs**: FR-009, FR-010, Section 9.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Directory `.github/skills/spec-requirements/` exists
  - [x] Directory `.github/skills/spec-user-stories/` exists
  - [x] Directory `.github/skills/spec-data-model/` exists
  - [x] Directory `.github/skills/spec-api-design/` exists
  - [x] Directory `.github/skills/spec-architecture/` exists
  - [x] Directory `.github/skills/spec-security/` exists
  - [x] Directory `.github/skills/spec-test-strategy/` exists
  - [x] Directory `.github/skills/spec-traceability/` exists
  - [x] All directory names match FR-010 canonical list verbatim
- **Test requirements**: none (directory existence check)
- **Depends on**: none
- **Implementation Guidance**:
  - Use `mkdir -p` or equivalent to create directories
  - Names are case-sensitive and hyphenated: `spec-requirements`, not `specRequirements`
  - Reference: existing pattern in `.github/skills/review-*/` directories
  - The glob pattern `.github/skills/spec-*/SKILL.md` must match all 8

### T08-02 - Create stub SKILL.md files

- **Description**: Create a minimal stub SKILL.md in each of the 8 skill directories. Each stub must have valid YAML frontmatter with `name` and `description` fields so the coordinator's dynamic discovery (FR-009) can parse them. The body should contain a placeholder indicating the skill is not yet implemented.
- **Spec refs**: FR-009, FR-023, Section 7.4
- **Parallel**: Yes (with T08-01 completed)
- **Acceptance criteria**:
  - [x] Each skill directory contains a SKILL.md file
  - [x] Each SKILL.md has YAML frontmatter with `name: spec-<name>` matching its directory
  - [x] Each SKILL.md has a `description` field (1-500 characters)
  - [x] Each SKILL.md body contains a placeholder comment indicating "Not yet implemented"
  - [x] The coordinator's glob scan `.github/skills/spec-*/SKILL.md` returns exactly 8 results
- **Test requirements**: none (file existence and frontmatter validation)
- **Depends on**: T08-01
- **Implementation Guidance**:
  - Follow the existing skill file pattern from `.github/skills/review-spec/SKILL.md`
  - YAML frontmatter structure:
    ```yaml
    ---
    name: spec-<name>
    description: "<one-line purpose>"
    ---
    ```
  - Body placeholder: `# spec-<name> - [Not Yet Implemented]\n\nThis skill will be implemented in WP<NN>.`
  - Description text per skill:
    - spec-requirements: "Produces functional requirements, NFRs, constraints, and out-of-scope sections"
    - spec-user-stories: "Produces user stories, acceptance scenarios, edge cases, and user flows"
    - spec-data-model: "Produces data model with typed entities and companion artifact files"
    - spec-api-design: "Produces API/interface design with typed contracts and error catalog artifacts"
    - spec-architecture: "Produces system architecture, tech stack, directory structure, and config schema artifact"
    - spec-security: "Produces expanded security requirements with OWASP mitigations"
    - spec-test-strategy: "Produces test requirements with BDD scenarios mapped to acceptance criteria"
    - spec-traceability: "Produces traceability matrix, glossary, references, and version history"

### T08-03 - Create spec-patterns.md placeholder

- **Description**: Create `.sdd/reviews/spec-patterns.md` as a placeholder for domain-specific spec patterns. The coordinator reads this file (if it exists) before dispatching skills per FR-019. Initial content follows the same format as the existing `review-patterns.md`.
- **Spec refs**: FR-019, Section 9.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] File `.sdd/reviews/spec-patterns.md` exists
  - [x] File follows the same structure as `.sdd/reviews/review-patterns.md` (header, Active Patterns section, Resolved section)
  - [x] Active Patterns section starts empty ("(none)")
  - [x] File has a header comment explaining its purpose: patterns learned from spec generation to avoid repeating mistakes
- **Test requirements**: none (file existence)
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.sdd/reviews/review-patterns.md` for format
  - Template:
    ```markdown
    # Spec Domain Patterns

    Patterns learned from specification generation. The Spec Architect coordinator reads
    this file before dispatching skills (FR-019) to avoid repeating known mistakes.

    - Last updated: <date>
    - Last spec: (none)

    ## Active Patterns

    (none)

    ## Resolved

    (none)
    ```

### T08-04 - Document artifact directory convention

- **Description**: Add a README or convention note documenting the companion artifacts directory pattern: `.sdd/specs/artifacts/<NNN>-<idea-name>/`. This convention is used by the coordinator (FR-008) and all skills that produce artifacts (FR-014-016).
- **Spec refs**: FR-008, FR-014, FR-015, FR-016, Section 7.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] A file `.sdd/specs/artifacts/README.md` exists
  - [x] It documents the naming convention: `<NNN>-<idea-name>/` subdirectory per spec
  - [x] It lists the expected artifact files: `data-models.<ext>`, `state-machines.<ext>`, `api-contracts.<ext>`, `error-catalog.<ext>`, `interfaces.<ext>`, `config-schema.<ext>`
  - [x] It documents the manifest comment format (FR-028)
  - [x] It documents target language selection logic (FR-015): use spec's tech stack or default to TypeScript
- **Test requirements**: none (documentation)
- **Depends on**: none
- **Implementation Guidance**:
  - The `.sdd/specs/artifacts/` directory may not exist yet -- create it
  - Manifest comment template from FR-028:
    ```
    // Generated by: spec-<skill-name> skill
    // Source spec: .sdd/specs/<NNN>-<idea-name>.spec.md, Section <N>
    // Target language: <language>
    // DO NOT EDIT MANUALLY -- regenerated on spec revision
    ```
  - File extensions by language: `.ts` (TypeScript), `.py` (Python), `.sql` (SQL DDL)

### T08-05 - Define common skill contract template

- **Description**: Create a reference document (or section in a README) that defines the common input/output contract all spec skills must follow (FR-023 through FR-028). This serves as the "spec skill developer guide" for creating new skills.
- **Spec refs**: FR-023, FR-024, FR-025, FR-026, FR-027, FR-028
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] A document exists that specifies the 8 inputs every spec skill receives (skill_path, accumulator_path, artifacts_dir, brief_path, research_summary, section_numbers, patterns, target_language)
  - [x] It specifies the 5-step execution sequence (read SKILL.md, read accumulator, read brief, write section, produce artifacts)
  - [x] It specifies the output format (FR-025): numbered headings, FR-XXX identifiers, SHALL statements, implementation contracts
  - [x] It specifies the constraint: no modification of prior sections (FR-026), use `[CROSS-REF ISSUE]` markers instead
  - [x] It specifies the constraint: no modification of coordinator sections 1-3 (FR-027)
  - [x] It specifies the manifest comment format for artifact files (FR-028)
- **Test requirements**: none (documentation)
- **Depends on**: none
- **Implementation Guidance**:
  - This can be a section in `.github/skills/README.md` or a standalone `.github/skills/SPEC-SKILL-CONTRACT.md`
  - Reference: existing skill pattern from review skills
  - The Coder should place this where it is most discoverable for future skill authors

## Implementation Notes

- All tasks in this WP produce markdown files or empty directories. No executable code.
- The directories created by T08-01 are required by the coordinator's glob scan (FR-009). Without them, the coordinator halts with "no spec skills installed."
- Stub SKILL.md files (T08-02) are intentionally minimal. Full skill content is written in WP10-WP13.
- This WP is the prerequisite for all subsequent WPs (WP09-WP13).

## Parallel Opportunities

All tasks (T08-01 through T08-05) can be worked concurrently. None depend on each other except T08-02 depends on T08-01 (directories must exist before files can be placed in them).

## Risks & Mitigations

- **Risk**: Directory names don't exactly match the glob pattern used by the coordinator.
  - **Mitigation**: Copy names verbatim from FR-010. Verify with `ls .github/skills/spec-*/SKILL.md`.
- **Risk**: Stub SKILL.md files have invalid YAML frontmatter, causing coordinator discovery to fail.
  - **Mitigation**: Validate YAML after creation. Reference existing review skill files for format.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T16:30:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: N/A (scaffolding WP -- no executable code for review skills to analyze)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 39 acceptance criteria verified against implementation
- [PASS] Activity Log: Consistent lane transitions (planned -> doing -> for_review)
- [WARN] Commit granularity: All 5 tasks in a single commit (acceptable for scaffolding)
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. No action items required.

(none)

### Warnings
- [WARN] PROC-003: All 5 tasks (T08-01 through T08-05) committed in a single commit rather than one per task. Acceptable for a pure scaffolding WP with only file/directory creation.

### Cross-Correlation Notes
No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| Spec FR Verification | 14 | 0 | 0 |
| Task Verification (T08-01 to T08-05) | 5 | 0 | 0 |
| Encoding | 1 | 0 | 0 |
| **Total** | **23** | **1** | **0** |

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T16:00:00Z - coder - lane=doing - Starting implementation of T08-01 through T08-05
- 2026-04-05T16:15:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T16:30:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARN)
