---
lane: done
docs_completed: true
---

# WP47 - Pattern File Propagation

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P3 |
| Lane | planned |
| Depends on | WP40 |
| Goal | Enable mid-cycle pattern propagation by adding version tracking to pattern files and version-check-before-dispatch to coordinators |
| Status | Complete |
| Independent Test | Have the Review Coordinator add a pattern to code-patterns.md and increment patterns_version. Dispatch the Coder for the next skill. Verify the Coder detects the version change and reloads patterns before dispatch. |
| Parallelisable | Yes (with WP46, WP48) |
| Prompt | `.sdd/plans/WP47-pattern-propagation.md` |

## Objective

Add `patterns_version` frontmatter to all 4 domain pattern files, update the Review Coordinator to increment this version when it modifies patterns, and update all 4 coordinator agents (Spec Architect, Planner, Coder, Docs Agent) to check `patterns_version` before each skill dispatch and reload if changed. This enables mid-cycle pattern propagation so newly discovered patterns take effect within the same pipeline run.

## Spec References

FR-052, FR-053, FR-054, Section 4.11 (Pattern File Propagation), Section 7.5 (Pattern File Frontmatter Extended), Section 8.4 (Pattern File Read Interface), US-12

## Tasks

### T47-01 - Add patterns_version to spec-patterns.md

- **Description**: Add `patterns_version: 1` to the YAML frontmatter of `.sdd/reviews/spec-patterns.md`.
- **Spec refs**: FR-052, Section 7.5
- **Parallel**: Yes (with T47-02, T47-03, T47-04)
- **Acceptance criteria**:
  - [x] spec-patterns.md has `patterns_version: 1` in YAML frontmatter (FR-052)
  - [x] Existing content is preserved
  - [x] `patterns_version` is a positive integer
- **Test requirements**: content (YAML frontmatter parse)
- **Depends on**: none
- **Implementation Guidance**:
  - Add the field to the existing YAML frontmatter block (between `---` delimiters)
  - If no frontmatter exists, create one
  - Files to modify: `.sdd/reviews/spec-patterns.md`

### T47-02 - Add patterns_version to plan-patterns.md

- **Description**: Add `patterns_version: 1` to the YAML frontmatter of `.sdd/reviews/plan-patterns.md`.
- **Spec refs**: FR-052, Section 7.5
- **Parallel**: Yes (with T47-01, T47-03, T47-04)
- **Acceptance criteria**:
  - [x] plan-patterns.md has `patterns_version: 1` in YAML frontmatter (FR-052)
  - [x] Existing content is preserved
- **Test requirements**: content (YAML frontmatter parse)
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.sdd/reviews/plan-patterns.md`

### T47-03 - Add patterns_version to code-patterns.md

- **Description**: Add `patterns_version: 1` to the YAML frontmatter of `.sdd/reviews/code-patterns.md`.
- **Spec refs**: FR-052, Section 7.5
- **Parallel**: Yes (with T47-01, T47-02, T47-04)
- **Acceptance criteria**:
  - [x] code-patterns.md has `patterns_version: 1` in YAML frontmatter (FR-052)
  - [x] Existing content is preserved
- **Test requirements**: content (YAML frontmatter parse)
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.sdd/reviews/code-patterns.md`

### T47-04 - Add patterns_version to doc-patterns.md

- **Description**: Add `patterns_version: 1` to the YAML frontmatter of `.sdd/reviews/doc-patterns.md`.
- **Spec refs**: FR-052, Section 7.5
- **Parallel**: Yes (with T47-01, T47-02, T47-03)
- **Acceptance criteria**:
  - [x] doc-patterns.md has `patterns_version: 1` in YAML frontmatter (FR-052)
  - [x] Existing content is preserved
- **Test requirements**: content (YAML frontmatter parse)
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.sdd/reviews/doc-patterns.md`

### T47-05 - Update Review Coordinator to increment patterns_version

- **Description**: Add logic to the Review Coordinator to increment `patterns_version` by 1 each time it adds, modifies, or retires a pattern in a domain patterns file.
- **Spec refs**: FR-053, Section 4.11
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Review Coordinator SHALL increment `patterns_version` by 1 each time it modifies a patterns file (FR-053)
  - [x] If `patterns_version` is missing, the Review Coordinator SHALL add it with value 1 (FR-053)
  - [x] The increment applies to add, modify, and retire operations
  - [x] Given the Review Coordinator adds a pattern and increments patterns_version, the version in the file increases by 1 (US-12 Scenario 1)
- **Test requirements**: BDD (US-12 Scenario 1)
- **Depends on**: T47-01 through T47-04
- **Implementation Guidance**:
  - Find the pattern curation/management section in review-coordinator.agent.md
  - Add: "After modifying any patterns file, increment `patterns_version` in that file's frontmatter by 1"
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T47-06 - Update coordinator agents for version-check-before-dispatch

- **Description**: Update the 4 coordinator agents (Spec Architect, Planner, Coder, Docs Agent) to record `patterns_version` when first reading patterns and re-read the file before each skill dispatch if the version has changed.
- **Spec refs**: FR-054, Section 4.11, Section 8.4
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator agents SHALL record `patterns_version` when first reading the patterns file (FR-054)
  - [x] Before each skill dispatch, coordinators SHALL check if `patterns_version` has changed (FR-054)
  - [x] If version has changed, coordinator re-reads the file and uses updated patterns (FR-054)
  - [x] If the patterns file is unreadable on re-check, the coordinator SHALL use last cached patterns and log a warning (FR-054)
  - [x] If frontmatter is missing, treating `patterns_version` as 0 triggers a reload every time (safe default)
  - [x] Given the Coder coordinator dispatches the next skill after a version change, it detects the change and reloads (US-12 Scenario 1)
  - [x] Given the patterns file is unreadable, the coordinator uses cached patterns (US-12 Scenario 2)
- **Test requirements**: BDD (US-12 Scenario 1, Scenario 2)
- **Depends on**: T47-05
- **Implementation Guidance**:
  - For each coordinator, find the patterns consumption section
  - Add before pre-dispatch check: "Read `patterns_version` from the patterns file frontmatter. If it differs from the last recorded version, re-read the full patterns content."
  - Error E-031 (PATTERNS_UNREADABLE): use cached patterns, log warning
  - Error E-032 (PATTERNS_VERSION_INVALID): treat as 0, always reload
  - Files to modify: `.github/agents/spec-architect.agent.md`, `.github/agents/planner.agent.md`, `.github/agents/coder.agent.md`, `.github/agents/docs-agent.agent.md`

## Implementation Notes

- All deliverables are markdown file updates -- no executable code
- This is a polling mechanism (Design Decision 5, Section 9.4): coordinators check the version counter before each skill dispatch
- Pattern propagation only matters for long pipeline runs where the Review Coordinator finds patterns mid-cycle
- Short-circuit: if the patterns file has not changed, no re-read occurs (just a version comparison)
- The 4 pattern files already exist from WP28 (Spec 006). This WP only adds frontmatter and updates agent logic

## Risks & Mitigations

- **Risk**: Pattern files may not have frontmatter yet. **Mitigation**: Both T47-01-04 add frontmatter, and the agent logic handles missing frontmatter (treat version as 0).
- **Risk**: Coordinators may have different patterns consumption implementations. **Mitigation**: Add the version check logic consistently to all 4 coordinators using the same template.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-07T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-07T00:00:01Z - coder - T47-01,T47-02,T47-03,T47-04 completed - Added patterns_version: 1 frontmatter to all 4 domain pattern files
- 2026-04-07T00:00:02Z - coder - T47-05 completed - Added patterns_version increment logic to Review Coordinator
- 2026-04-07T00:00:03Z - coder - T47-06 completed - Added version-check-before-dispatch to all 4 coordinators
- 2026-04-07T00:00:04Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-07T00:00:05Z - review-coordinator - lane=done - Verdict: Approved
- 2026-04-07T00:00:06Z - docs-agent - docs-complete - Documentation generated for WP47
- 2026-04-07T12:00:00Z - review-coordinator - lane=done - Verdict: Approved (re-review round 2, no changes since round 1)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-07T12:00:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS), review-security (N/A), review-quality (PASS), review-tests (N/A), review-architecture (PASS), review-performance (N/A), review-docs (N/A), review-deps (N/A)
> **Review round**: 2

### Process Compliance
- [PASS] Spec Compliance Checklist: All 18 acceptance criteria checked and verified against FR-052, FR-053, FR-054
- [PASS] Activity Log: Proper lane transitions (planned -> doing -> for_review -> done)
- [PASS] Commit granularity: 3 commits for 6 tasks (T47-01-04 grouped as parallel tasks, T47-05 separate, T47-06 separate)
- [PASS] Encoding: No violations found

### Review Feedback

No FAIL findings. No action required.

### Warnings

No warnings.

### Cross-Correlation Notes

No cross-correlation findings. Re-review confirmed no files modified since round 1 approval.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 3 | 0 | 0 |
| review-security | 0 | 0 | 0 |
| review-quality | 8 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 3 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 0 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **18** | **0** | **0** |
