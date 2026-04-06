---
lane: done
---

# WP46 - Schema Versioning Protocol

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P2 |
| Lane | planned |
| Depends on | WP42 |
| Goal | Add version_history sections to all schema files and document the schema versioning protocol |
| Status | Not Started |
| Independent Test | Open any schema file in `.github/schemas/` and verify it contains a `version_history` array with at least one entry. Read the developer guide and verify it contains a "Schema Versioning Protocol" section. |
| Parallelisable | Yes (with WP44, WP45) |
| Prompt | `.sdd/plans/WP46-schema-versioning.md` |

## Objective

Add version_history sections to every handoff schema file in `.github/schemas/` (including the new return schemas from WP42) and document the versioning protocol (major bump for breaking changes, same version for additive changes) in the developer guide. Also add path placeholder regex patterns to schemas that use placeholders.

## Spec References

FR-044, FR-045, FR-046, FR-047, FR-048, Section 4.9 (Schema Versioning Protocol), Section 7.3 (Handoff Schema), US-10

## Tasks

### T46-01 - Add version_history to existing forward schemas

- **Description**: Add a `version_history` YAML array to each existing forward handoff schema file. Each initial entry has version "handoff/v1", the current date, and description "Initial version."
- **Spec refs**: FR-044, FR-045, Section 7.3
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each existing handoff schema in `.github/schemas/` contains a `version_history` section (FR-044)
  - [x] Each version_history entry contains: `version` (string), `date` (ISO-8601), `description` (string) (FR-045)
  - [x] At least one entry exists per schema (the initial version)
  - [x] Given a maintainer opens coder-to-reviewer.schema.yaml, they find a version_history array (US-10 Scenario 1)
- **Test requirements**: content (YAML parse), BDD (US-10 Scenario 1)
- **Depends on**: none
- **Implementation Guidance**:
  - Add version_history at the end of each schema file, after validation_rules
  - Existing schemas: ideation-to-spec, spec-to-planner, planner-to-coder, coder-to-reviewer, reviewer-to-coder, reviewer-to-spec, planner-to-spec, orchestrator-handoff
  - Files to modify: all `.github/schemas/*.schema.yaml` (existing forward schemas)

### T46-02 - Add version_history to new schemas

- **Description**: Add `version_history` to the 3 return schemas and the base schema created by WP42.
- **Spec refs**: FR-044, FR-045
- **Parallel**: Yes (with T46-01)
- **Acceptance criteria**:
  - [x] reviewer-to-orchestrator.schema.yaml has version_history with initial entry
  - [x] coder-complete-to-orchestrator.schema.yaml has version_history with initial entry
  - [x] docs-agent-to-orchestrator.schema.yaml has version_history with initial entry
  - [x] base-handoff.schema.yaml has version_history with initial entry
- **Test requirements**: content (YAML parse)
- **Depends on**: none (WP42 must be complete, enforced by WP-level dependency)
- **Implementation Guidance**:
  - Same format as T46-01
  - Base schema uses version "base/v1" instead of "handoff/v1"
  - Files to modify: 4 new schema files from WP42

### T46-03 - Add path placeholder regex patterns

- **Description**: Add `placeholder_patterns` section to schemas that use path placeholders ({NNN}, {name}, {slug}, {NN}) with YAML comments showing the regex pattern for each.
- **Spec refs**: FR-048, Section 7.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schemas with path placeholders have `placeholder_patterns` section (FR-048)
  - [x] `{NNN}` has pattern `\d{2,3}` (FR-048)
  - [x] `{name}` has pattern `[a-z0-9-]+` (FR-048)
  - [x] `{slug}` has pattern `[a-z0-9-]+` (FR-048)
  - [x] `{NN}` has pattern `\d{2}` (FR-048)
  - [x] If a placeholder does not match its regex during validation, the agent halts with descriptive error
- **Test requirements**: content (YAML parse)
- **Depends on**: none
- **Implementation Guidance**:
  - Add placeholder_patterns as a new top-level section in relevant schemas
  - Only schemas with artifact path patterns need this section
  - Error behavior: halt with "Artifact path '<path>' does not match expected pattern for placeholder '<placeholder>'"
  - Files to modify: schemas that contain artifact path patterns (spec-to-planner, planner-to-coder, coder-to-reviewer at minimum)

### T46-04 - Document versioning protocol in developer guide

- **Description**: Add a "Schema Versioning Protocol" section to `.sdd/docs/developer-guide.md` documenting when to increment schema versions and when to keep them unchanged.
- **Spec refs**: FR-046, FR-047, NFR-013
- **Parallel**: Yes (with T46-01, T46-02, T46-03)
- **Acceptance criteria**:
  - [x] Developer guide contains a "Schema Versioning Protocol" section (NFR-013)
  - [x] Breaking changes (removing fields, changing types, removing enum values, renaming fields) require version increment from handoff/vN to handoff/v(N+1) (FR-046)
  - [x] Additive changes (new optional fields, new enum values, new optional rules) retain current version (FR-047)
  - [x] Both breaking and additive changes require a version_history entry
  - [x] Given a maintainer removes a required field, they know to increment the version (US-10 Scenario 2)
  - [x] Given a maintainer adds an optional field, they know to keep the version (US-10 Scenario 3)
- **Test requirements**: content (grep search), BDD (US-10 Scenario 2, Scenario 3)
- **Depends on**: none
- **Implementation Guidance**:
  - The protocol is documentation-only -- no runtime enforcement
  - Include examples of breaking vs additive changes
  - Files to modify: `.sdd/docs/developer-guide.md`

### T46-05 - Verify version_history completeness

- **Description**: Verify every schema file in `.github/schemas/` has a `version_history` section. Check that all 11+ schemas are covered.
- **Spec refs**: FR-044, Section 11.1
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [x] All handoff schema files in `.github/schemas/` have version_history sections
  - [x] No schema file is missing version_history (grep verification)
  - [x] version_history entries are ordered chronologically (newest last)
- **Test requirements**: content (grep search across all schema files)
- **Depends on**: T46-01, T46-02
- **Implementation Guidance**:
  - Use grep to search for `version_history` across all .schema.yaml files
  - Verify count matches total schema file count

## Implementation Notes

- All deliverables are YAML schema file updates and markdown documentation -- no executable code
- Depends on WP42 because the return schemas and base schema must exist before adding version_history to them
- The versioning protocol is a convention enforced by review, not by runtime validation (C-05)
- Missing version_history is a warning, not an error (FR-044): graceful degradation for pre-hardening schemas
- The developer guide section ensures future maintainers follow the protocol without needing this spec

## Risks & Mitigations

- **Risk**: Some schema files may have different structures that make it awkward to add version_history. **Mitigation**: Use a consistent insertion point (end of file, after validation_rules).
- **Risk**: Placeholder regex patterns may not cover all edge cases. **Mitigation**: Use exact regex patterns from the spec (FR-048).

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-07T00:10:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (PASS), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (PASS), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 18 acceptance criteria checked and verified against implementation
- [PASS] Activity Log: Consistent lane transitions (planned -> doing -> for_review)
- [PASS] Commit granularity: 4 implementation commits + 1 submission commit matching task structure
- [PASS] Encoding: No prohibited Unicode characters found

### Review Feedback

No FAIL findings. No action required.

### Warnings

No warnings.

### Cross-Correlation Notes

No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 6 | 0 | 0 |
| review-security | 0 | 0 | 0 |
| review-quality | 4 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 2 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 3 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **19** | **0** | **0** |

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-07T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-07T00:01:00Z - coder - T46-01 completed - Added version_history to 8 existing forward schemas
- 2026-04-07T00:01:00Z - coder - T46-02 completed - Added version_history to 4 new schemas (base, 3 return schemas)
- 2026-04-07T00:02:00Z - coder - T46-03 completed - Added placeholder_patterns to 10 schemas with path placeholders
- 2026-04-07T00:03:00Z - coder - T46-04 completed - Added Schema Versioning Protocol section to developer guide
- 2026-04-07T00:04:00Z - coder - T46-05 completed - Verified all 12 schema files have version_history
- 2026-04-07T00:05:00Z - coder - lane=for_review - All tasks complete, verification passing
- 2026-04-07T00:10:00Z - review-coordinator - lane=done - Verdict: Approved
