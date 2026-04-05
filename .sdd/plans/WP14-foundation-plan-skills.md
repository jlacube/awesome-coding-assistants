---
lane: done
---

# WP14 - Foundation: Plan Skill Scaffolding

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/003-planner-v2.spec.md` |
| Priority | P0 |
| Lane | planned |
| Depends on | none |
| Goal | Create the directory structure, stub skill files, and common plan-skill contract so Phase 1 and Phase 2 skills can be implemented |
| Status | Complete |
| Independent Test | Verify: 8 plan skill directories exist under `.github/skills/plan-*/`, each contains a stub `SKILL.md` with valid YAML frontmatter, and `PLAN-SKILL-CONTRACT.md` defines the 9-input contract |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP14-foundation-plan-skills.md` |

## Objective

Establish the foundation directories, stub files, and shared contract documentation required by all 8 planning skills and the Planner coordinator. This WP creates no functional behavior -- it builds the scaffolding that WP15-WP19 implement against. Mirrors the pattern established in WP08 (Spec Architect foundation).

## Spec References

FR-023, FR-024, FR-025, FR-026, FR-027, FR-039, Section 9.3 (Directory Structure), Section 9.2 (Technology Stack)

## Tasks

### T14-01 - Create plan skill directories

- **Description**: Create 8 directories under `.github/skills/` matching the canonical skill names from FR-011: `plan-decomposition`, `plan-acceptance`, `plan-interface-contracts`, `plan-data-schemas`, `plan-api-contracts`, `plan-state-machines`, `plan-error-catalogs`, `plan-cross-wp-validation`.
- **Spec refs**: FR-010, FR-011, Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] All 8 directories exist under `.github/skills/`
  - [x] Directory names match the canonical names from FR-011 exactly
  - [x] No extra directories created beyond the 8 specified
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Official docs: https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview
  - Pattern: Follow the same structure as existing `spec-*` skill directories
  - Known pitfalls: Directory names must match the glob pattern `.github/skills/plan-*/SKILL.md` used in FR-010 for dynamic discovery
  - Spec validation rules: Names are lowercase-kebab-case with `plan-` prefix

### T14-02 - Create stub SKILL.md files with YAML frontmatter

- **Description**: Create a stub `SKILL.md` in each of the 8 plan skill directories. Each stub SHALL have valid YAML frontmatter with `name`, `description`, and `argument-hint` fields. The body SHALL contain a placeholder comment indicating the skill is not yet implemented.
- **Spec refs**: FR-023, FR-024, Section 9.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each of the 8 directories contains a `SKILL.md` file
  - [x] Each `SKILL.md` has valid YAML frontmatter with `name`, `description`, and `argument-hint`
  - [x] `argument-hint` reads "Invoked by Planner Coordinator - do not call directly"
  - [x] The `name` field matches the directory name (e.g., `plan-decomposition`)
  - [x] The `description` field matches the canonical purpose from FR-011
- **Test requirements**: none
- **Depends on**: T14-01
- **Implementation Guidance**:
  - Pattern: Mirror the frontmatter format from `.github/skills/spec-requirements/SKILL.md`:
    ```yaml
    ---
    name: plan-decomposition
    description: "WP identification, task breakdown, sequencing, and dependencies"
    argument-hint: "Invoked by Planner Coordinator - do not call directly"
    ---
    ```
  - Known pitfalls: YAML frontmatter must start on line 1 with `---` and end with `---`. No leading whitespace.

### T14-03 - Create PLAN-SKILL-CONTRACT.md

- **Description**: Create `.github/skills/PLAN-SKILL-CONTRACT.md` defining the common input/output contract for all plan-* skills. This contract parallels `SPEC-SKILL-CONTRACT.md` but with the 9-input spec from FR-023 and the two-phase output rules from FR-025/FR-026.
- **Spec refs**: FR-023, FR-024, FR-025, FR-026, FR-027
- **Parallel**: No
- **Acceptance criteria**:
  - [x] File exists at `.github/skills/PLAN-SKILL-CONTRACT.md`
  - [x] Contract specifies all 9 inputs from FR-023: skill_path, plan_dir, contracts_dir, spec_path, spec_artifacts_dir, research_summary, target_language, patterns, phase
  - [x] Contract specifies the 4-step execution sequence from FR-024: read SKILL.md, read plan state, read spec + artifacts, write assigned artifacts
  - [x] Contract specifies Phase 1 output rules (FR-025): write WP files and README to `.sdd/plans/`
  - [x] Contract specifies Phase 2 output rules (FR-026): write contract files to `.sdd/plans/contracts/<WP-slug>/`
  - [x] Contract specifies modification constraint (FR-027): skills SHALL NOT modify files written by earlier skills unless marked `[CONSISTENCY FIX: <description>]`
  - [x] No em dashes, smart quotes, or curly apostrophes (ASCII only)
- **Test requirements**: none
- **Depends on**: T14-01
- **Implementation Guidance**:
  - Pattern: Model after `.github/skills/SPEC-SKILL-CONTRACT.md` structure (inputs table, execution sequence, output format, modification constraints)
  - Known pitfalls: The plan contract has 9 inputs (vs 8 for spec contract) -- the extra input is `phase` (1 or 2)
  - FR-027 specifies the `[CONSISTENCY FIX: <description>]` marker format exactly

### T14-04 - Create contracts directory structure

- **Description**: Create the `.sdd/plans/contracts/` directory and a `shared/` subdirectory within it. These directories hold Phase 2 contract output files scoped per WP and shared config.
- **Spec refs**: FR-017, FR-026, FR-052, Section 9.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Directory `.sdd/plans/contracts/` exists
  - [x] Directory `.sdd/plans/contracts/shared/` exists
  - [x] A `.gitkeep` file is placed in each empty directory to ensure git tracking
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Standard git practice -- use `.gitkeep` to track empty directories
  - These directories will be populated by Phase 2 skills (WP17-WP19)

### T14-05 - Define manifest header template

- **Description**: Document the contract file manifest header template in `PLAN-SKILL-CONTRACT.md` (as a subsection). The template SHALL match FR-039 exactly, with parameterized fields for skill name, spec path, WP slug, and target language.
- **Spec refs**: FR-039
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Manifest header template is documented in `PLAN-SKILL-CONTRACT.md`
  - [x] Template includes all 5 fields: `Generated by`, `Source spec`, `Work package`, `Target language`, `DO NOT EDIT MANUALLY` warning
  - [x] Template matches FR-039 format exactly
- **Test requirements**: none
- **Depends on**: T14-03
- **Implementation Guidance**:
  - FR-039 specifies the exact format:
    ```
    // Generated by: <skill-name> skill
    // Source spec: .sdd/specs/<NNN>-<name>.spec.md
    // Work package: WP<NN>-<slug>
    // Target language: <language>
    // DO NOT EDIT MANUALLY -- regenerated on plan revision
    ```

### T14-06 - Verify encoding compliance

- **Description**: Run an automated check on all created files to verify no prohibited Unicode characters (em dashes U+2013/U+2014, smart quotes U+201C/U+201D/U+2018/U+2019, curly apostrophes) are present. Fix any violations.
- **Spec refs**: Section 9.2 (plain ASCII encoding requirement inherited from pipeline conventions)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Python script confirms zero prohibited Unicode characters across all new files
  - [x] All hyphens are ASCII `-` (U+002D), all quotes are straight `"` or `'`
  - [x] Script output shows clean results
- **Test requirements**: unit (encoding validation script)
- **Depends on**: T14-01, T14-02, T14-03, T14-04, T14-05
- **Implementation Guidance**:
  - Reuse the encoding check pattern from WP08:
    ```python
    python -c "
    import re, glob
    bad_chars = {'\u2013': 'en-dash', '\u2014': 'em-dash', ...}
    for f in glob.glob('.github/skills/plan-*/**', recursive=True): ...
    "
    ```

## Implementation Notes

- This WP mirrors WP08 (Spec Architect foundation) in structure and purpose
- All 8 skill stubs are intentionally non-functional -- they exist to enable dynamic skill discovery (FR-010) and to be fleshed out in WP16-WP19
- The `PLAN-SKILL-CONTRACT.md` serves the same role as `SPEC-SKILL-CONTRACT.md`: a shared reference that all skill implementations follow

## Parallel Opportunities

- T14-04 can run in parallel with T14-01/T14-02/T14-03
- T14-05 depends on T14-03 but can run in parallel with T14-04

## Risks & Mitigations

- **Risk**: Directory naming mismatch with glob pattern in FR-010. **Mitigation**: Verify discovery works with a test glob scan after creation.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T10:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T10:15:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T18:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (1 WARN)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T18:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-quality (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked
- [PASS] Activity Log: Consistent lane transitions (planned -> doing -> for_review)
- [WARN] Commit granularity: Single bulk commit (267ea89) for all 6 tasks
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. No action required.

### Warnings
- [WARN] PROC-003: Single bulk commit covers all 6 tasks. Acceptable for scaffolding WP.

### Cross-Correlation Notes
- No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 10 | 0 | 0 |
| review-quality | 3 | 0 | 0 |
| **Total** | **16** | **1** | **0** |
