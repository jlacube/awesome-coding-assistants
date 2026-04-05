---
lane: doing
---

# WP28 - Domain-Specific Pattern Files & Migration

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/006-handoff-schemas-patterns.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | Four domain-specific pattern files exist in `.sdd/reviews/` with correct format, and legacy review-patterns.md is migrated |
| Status | Not Started |
| Independent Test | Verify 4 files exist: `spec-patterns.md`, `plan-patterns.md`, `code-patterns.md`, `doc-patterns.md` in `.sdd/reviews/`; verify legacy `review-patterns.md` is renamed to `.bak`; verify all pattern IDs use `PAT-{DOMAIN}-XXX` format |
| Parallelisable | Yes (with WP27) |
| Prompt | `.sdd/plans/WP28-domain-specific-pattern-files.md` |

## Objective

Create and standardize domain-specific pattern files so each pipeline agent reads only its own domain's patterns. Migrate entries from the legacy single `review-patterns.md` into the appropriate domain files. This WP delivers the pattern file artifacts; agent integration happens in WP29.

## Spec References

FR-008, FR-009, FR-010, FR-016, Section 7.2, Section 9.1

## Tasks

### T28-01 - Create plan-patterns.md with FR-009 structure

- **Description**: Create a new `plan-patterns.md` file in `.sdd/reviews/` following the exact pattern file format from FR-009. Include the "Active Patterns" and "Retired Patterns" sections. This file is consumed by the Planner agent.
- **Spec refs**: FR-008 (item 2), FR-009
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] File exists at `.sdd/reviews/plan-patterns.md`
  - [x] File follows the exact structure from FR-009: title, "Active Patterns" section, "Retired Patterns" section
  - [x] File header is `# Plan Patterns`
  - [x] File contains at least the structural template (may be empty of actual patterns initially)
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Use the exact template from FR-009 (spec Section 4.2.2)
  - Domain: PLAN -- pattern IDs will use PAT-PLAN-XXX prefix
  - The file may start empty (no active patterns) -- the structure must be in place
  - Files to create: `.sdd/reviews/plan-patterns.md`

### T28-02 - Create doc-patterns.md with FR-009 structure

- **Description**: Create a new `doc-patterns.md` file in `.sdd/reviews/` following the exact pattern file format from FR-009. This file is consumed by the Docs Agent.
- **Spec refs**: FR-008 (item 4), FR-009
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] File exists at `.sdd/reviews/doc-patterns.md`
  - [x] File follows the exact structure from FR-009: title, "Active Patterns" section, "Retired Patterns" section
  - [x] File header is `# Doc Patterns`
  - [x] File contains at least the structural template
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Use the exact template from FR-009 (spec Section 4.2.2)
  - Domain: DOC -- pattern IDs will use PAT-DOC-XXX prefix
  - Files to create: `.sdd/reviews/doc-patterns.md`

### T28-03 - Update spec-patterns.md to FR-009 format

- **Description**: Update the existing `spec-patterns.md` in `.sdd/reviews/` to comply with the FR-009 structure. Ensure existing patterns have all required fields (id, title, status, added, source, trigger, prevention) and use the PAT-SPEC-XXX ID format per FR-010.
- **Spec refs**: FR-008 (item 1), FR-009, FR-010
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] `.sdd/reviews/spec-patterns.md` exists and follows FR-009 structure
  - [x] Every existing pattern entry has all required fields: id (PAT-SPEC-XXX), title, status, added, source, trigger, prevention
  - [x] Pattern IDs use the `PAT-SPEC-XXX` format per FR-010
  - [x] File has "Active Patterns" and "Retired Patterns" sections
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Read existing `.sdd/reviews/spec-patterns.md` first to understand current format
  - Add any missing required fields to existing entries
  - Rename any pattern IDs that do not follow PAT-SPEC-XXX format
  - Preserve existing pattern content -- only restructure, do not remove
  - Known pitfall: existing patterns may use different field names or formats; normalize to FR-009

### T28-04 - Update code-patterns.md to FR-009 format

- **Description**: Update the existing `code-patterns.md` in `.sdd/reviews/` to comply with the FR-009 structure. Ensure existing patterns have all required fields and use the PAT-CODE-XXX ID format per FR-010.
- **Spec refs**: FR-008 (item 3), FR-009, FR-010
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] `.sdd/reviews/code-patterns.md` exists and follows FR-009 structure
  - [x] Every existing pattern entry has all required fields: id (PAT-CODE-XXX), title, status, added, source, trigger, prevention
  - [x] Pattern IDs use the `PAT-CODE-XXX` format per FR-010
  - [x] File has "Active Patterns" and "Retired Patterns" sections
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Read existing `.sdd/reviews/code-patterns.md` first to understand current format
  - Add any missing required fields to existing entries
  - Rename any pattern IDs that do not follow PAT-CODE-XXX format
  - Preserve existing pattern content -- only restructure, do not remove

### T28-05 - Migrate legacy review-patterns.md to domain-specific files

- **Description**: Read all patterns from the legacy `.sdd/reviews/review-patterns.md`, categorize each into its domain (spec, plan, code, doc) based on content and trigger, write each pattern to the appropriate domain-specific file, and rename the legacy file to `review-patterns.md.bak`.
- **Spec refs**: FR-016
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The system SHALL read all patterns from `.sdd/reviews/review-patterns.md`
  - [ ] Each pattern SHALL be categorized into one of: spec, plan, code, doc based on content and trigger
  - [ ] Each categorized pattern SHALL be written to the correct domain-specific file
  - [ ] The legacy file SHALL be renamed to `review-patterns.md.bak`
  - [ ] If a pattern cannot be categorized, it SHALL be placed in `code-patterns.md` as the default domain with a note for manual review
  - [ ] If a pattern with the same ID already exists in the target domain file, the duplicate SHALL be skipped
- **Test requirements**: BDD (migration verification)
- **Depends on**: T28-01, T28-02, T28-03, T28-04
- **Implementation Guidance**:
  - Read `.sdd/reviews/review-patterns.md` and parse each pattern entry
  - Categorization heuristic: check trigger and prevention text for domain keywords (e.g., "spec", "FR", "requirement" -> SPEC; "WP", "task", "plan" -> PLAN; "code", "implementation", "function" -> CODE; "docs", "documentation" -> DOC)
  - Default domain for uncategorizable patterns: CODE (per spec edge case)
  - Write to domain files AFTER T28-01 through T28-04 have created/updated them
  - Rename: `mv review-patterns.md review-patterns.md.bak`
  - Known pitfall: existing review-patterns.md may have patterns that belong to agents not yet created (e.g., Docs Agent). Place those in doc-patterns.md.
  - Error handling: If review-patterns.md cannot be parsed, halt and report parse error (FR-016)

### T28-06 - Verify migration idempotency

- **Description**: Verify that running the migration when domain files already have patterns does not duplicate entries. Confirm that running the migration a second time produces no changes.
- **Spec refs**: FR-016
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] The migration SHALL be idempotent: running it when domain files already exist does not duplicate patterns
  - [ ] Pattern count in each domain file remains the same after a second migration run
  - [ ] No pattern ID appears more than once in any domain file
- **Test requirements**: none
- **Depends on**: T28-05
- **Implementation Guidance**:
  - After initial migration (T28-05), verify each domain-specific file has no duplicate pattern IDs
  - If legacy file was already renamed to .bak, the migration should detect this and skip
  - This is a verification task -- document the result in the Activity Log

### T28-07 - Verify pattern ID format compliance

- **Description**: Scan all 4 domain-specific pattern files and confirm every pattern ID follows the `PAT-{DOMAIN}-XXX` format where DOMAIN is SPEC, PLAN, CODE, or DOC.
- **Spec refs**: FR-010
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Every pattern in `spec-patterns.md` has an ID matching `PAT-SPEC-XXX`
  - [ ] Every pattern in `plan-patterns.md` has an ID matching `PAT-PLAN-XXX`
  - [ ] Every pattern in `code-patterns.md` has an ID matching `PAT-CODE-XXX`
  - [ ] Every pattern in `doc-patterns.md` has an ID matching `PAT-DOC-XXX`
  - [ ] No pattern files contain executable code or template expressions (NFR-003)
- **Test requirements**: none
- **Depends on**: T28-05
- **Implementation Guidance**:
  - Scan each file for pattern entries and verify ID format
  - Valid regex: `^PAT-(SPEC|PLAN|CODE|DOC)-\d{3}$`
  - NFR-003 compliance: files must be purely declarative Markdown, no executable expressions

## Implementation Notes

All deliverables are Markdown files. The existing `spec-patterns.md` and `code-patterns.md` files may already have content that needs format normalization rather than replacement. Read first, then update.

The legacy `review-patterns.md` currently has entries that the Review Coordinator references. After migration, the Review Coordinator reference must be updated (handled in WP29).

Migration order matters: create/update domain files first (T28-01 through T28-04), then migrate (T28-05), then verify (T28-06, T28-07).

## Parallel Opportunities

T28-01, T28-02, T28-03, and T28-04 can all run in parallel. T28-05 through T28-07 must be sequential.

## Risks & Mitigations

- **Risk**: Existing patterns in spec-patterns.md and code-patterns.md may not match FR-009 format. **Mitigation**: Read existing files before modifying; normalize structure while preserving content.
- **Risk**: Legacy review-patterns.md has patterns that are hard to categorize by domain. **Mitigation**: Default uncategorizable patterns to code-patterns.md with a manual review note (per spec edge case).
- **Risk**: Migration breaks existing Review Coordinator pattern reading. **Mitigation**: WP29 updates the Review Coordinator to read domain-specific files instead of the legacy file.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
