---
lane: doing
---

# WP40 - Enum Registry & Canonical Conventions

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | Create the central enum registry and standardize Activity Log format across all agents |
| Status | Not Started |
| Independent Test | Read `.github/schemas/enums.yaml` and verify it contains exactly 4 enum groups (lane, spec_status, pipeline_stage, review_status) with the canonical values, plus a conventions section with the Activity Log format template. Search all agent files for the enums.yaml reference comment. |
| Parallelisable | No (foundation for WP41-WP48) |
| Prompt | `.sdd/plans/WP40-enum-registry-conventions.md` |

## Objective

Create the central enum registry file at `.github/schemas/enums.yaml` that defines all pipeline-wide enumeration values (lane, spec_status, pipeline_stage, review_status) and the canonical Activity Log format. Then update all agent instruction files to reference enums.yaml as the authoritative source for enum values and to use the canonical log entry format. This WP is the foundation for the entire hardening pass -- subsequent WPs depend on the registry existing.

## Spec References

FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, FR-014, FR-015, FR-016, FR-017, FR-018, FR-019, FR-020, Section 7.2 (Enum Registry), Section 8.2 (Enum Registry Read Interface), Section 9.3 (Directory Structure), US-06, US-07

## Tasks

### T40-01 - Create enums.yaml with enum groups and conventions

- **Description**: Create `.github/schemas/enums.yaml` containing four enum groups (`lane`, `spec_status`, `pipeline_stage`, `review_status`) with their exact values from the spec, plus a `conventions` section containing the canonical Activity Log format template.
- **Spec refs**: FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, FR-020, Section 7.2
- **Parallel**: No (foundation for all subsequent tasks)
- **Acceptance criteria**:
  - [x] File exists at `.github/schemas/enums.yaml` and is valid YAML (FR-008)
  - [x] `lane` enum group defines exactly: planned, doing, for_review, to_do, done, blocked (FR-010)
  - [x] `spec_status` enum group defines exactly: Draft, Validated, Approved (FR-011)
  - [x] `pipeline_stage` enum group defines exactly: idle, ideation, specification, planning, implementation, review, documentation, complete (FR-012)
  - [x] `review_status` enum group defines exactly: pending, has_feedback, acknowledged, approved (FR-013)
  - [x] `conventions.activity_log_format` contains the template: `<ISO-8601-timestamp> - <agent-name> - <action> - <details>` (FR-020)
  - [x] No extra enum values beyond those specified in the spec
- **Test requirements**: content (YAML parse + value comparison)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the data-models.yaml companion artifact Section 7.2 for exact field structure
  - Each enum group SHALL be a top-level YAML key mapping to an array of string values
  - The conventions section SHALL be a separate top-level key with the activity_log_format as a nested string field
  - Error E-010 (ENUM_REGISTRY_NOT_FOUND): agents halt if this file is missing
  - Error E-013 (ENUM_PARSE_ERROR): agents halt if this file has malformed YAML
  - Files to create: `.github/schemas/enums.yaml`

### T40-02 - Update Planner agent: remove "Final" reference, add enum reference

- **Description**: Edit `.github/agents/planner.agent.md` to remove any references to "Final" as a valid spec status value. Add a comment citing enums.yaml as the authoritative source for spec_status values.
- **Spec refs**: FR-014, FR-015, Section 4.2
- **Parallel**: Yes (independent of T40-03 through T40-07)
- **Acceptance criteria**:
  - [x] No references to "Final" as a spec status exist in the Planner agent file (FR-015)
  - [x] The Planner agent file contains a comment: `<!-- Enum source: .github/schemas/enums.yaml -->` near its enum references (FR-014)
  - [x] Spec status validation only accepts Draft, Validated, Approved
- **Test requirements**: content (grep search for "Final", grep for enum comment)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - Search for "Final" in the Planner and replace with the registry-backed values (Draft, Validated, Approved)
  - The comment should appear near where the Planner checks spec status (Step 1 of the workflow)
  - Files to modify: `.github/agents/planner.agent.md`

### T40-03 - Update Orchestrator agent: add enum references

- **Description**: Add enums.yaml reference comments to `.github/agents/orchestrator.agent.md` near where it references lane values, pipeline_stage values, and spec_status values.
- **Spec refs**: FR-014, Section 4.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Orchestrator agent file contains `<!-- Enum source: .github/schemas/enums.yaml -->` comment near enum references (FR-014)
  - [x] All inline lane value lists in the Orchestrator reference or match the registry values
  - [x] All inline pipeline_stage value lists match the registry values
- **Test requirements**: content (grep search)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - Look for sections that list lane values (planned, doing, for_review, etc.) and add the enum comment nearby
  - Look for sections that list pipeline stages and add the enum comment
  - Do not change the values themselves -- just add the reference comment
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T40-04 - Update Coder agent: enum references and canonical log format

- **Description**: Add enums.yaml reference comment and update the Activity Log Protocol section to use the canonical format `<ISO-8601> - coder - <action> - <details>`.
- **Spec refs**: FR-014, FR-016, FR-017, Section 4.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Coder agent file contains `<!-- Enum source: .github/schemas/enums.yaml -->` comment (FR-014)
  - [x] Activity Log Protocol section specifies the canonical format: `<ISO-8601-timestamp> - <agent-name> - <action> - <details>` (FR-016, FR-017)
  - [x] Agent name in log entries is `coder` (lowercase with no spaces)
  - [x] Format matches the canonical definition in enums.yaml conventions
- **Test requirements**: content (grep search)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - Locate the existing Activity Log Protocol section in coder.agent.md
  - Replace any existing format specification with the canonical format from FR-016
  - The canonical format uses ` - ` (space-hyphen-space) as separator between fields (NFR-008)
  - Files to modify: `.github/agents/coder.agent.md`

### T40-05 - Update Review Coordinator agent: enum references and canonical log format

- **Description**: Add enums.yaml reference comment and update any Activity Log format instructions to use the canonical format.
- **Spec refs**: FR-014, FR-016, FR-018, Section 4.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Review Coordinator agent file contains `<!-- Enum source: .github/schemas/enums.yaml -->` comment (FR-014)
  - [x] Activity Log entries use the canonical format: `<ISO-8601-timestamp> - review-coordinator - <action> - <details>` (FR-016, FR-018)
  - [x] Agent name in log entries is `review-coordinator` (lowercase with hyphens)
- **Test requirements**: content (grep search)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - Locate Activity Log format references in review-coordinator.agent.md
  - Ensure the format matches the canonical template exactly
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T40-06 - Update Docs Agent: enum references and canonical log format

- **Description**: Add enums.yaml reference comment and add or update the Activity Log format to use the canonical format.
- **Spec refs**: FR-014, FR-016, FR-019, Section 4.3
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Docs Agent file contains `<!-- Enum source: .github/schemas/enums.yaml -->` comment (FR-014)
  - [x] Activity Log entries use the canonical format: `<ISO-8601-timestamp> - docs-agent - <action> - <details>` (FR-016, FR-019)
  - [x] Agent name in log entries is `docs-agent` (lowercase with hyphen)
- **Test requirements**: content (grep search)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - If the Docs Agent does not yet have an Activity Log Protocol section, add one
  - Files to modify: `.github/agents/docs-agent.agent.md`

### T40-07 - Update Spec Architect agent: add enum reference

- **Description**: Add enums.yaml reference comment to the Spec Architect agent file near where it references spec_status values.
- **Spec refs**: FR-014, Section 4.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] Spec Architect agent file contains `<!-- Enum source: .github/schemas/enums.yaml -->` comment (FR-014)
  - [ ] Spec status references align with the registry values (Draft, Validated, Approved)
- **Test requirements**: content (grep search)
- **Depends on**: T40-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/spec-architect.agent.md`

## Implementation Notes

- All deliverables are YAML and markdown files -- no executable code
- enums.yaml is the foundational registry for the entire hardening pass. All subsequent WPs depend on it
- The Activity Log format standardization (FR-016-020) is bundled here because the canonical format template lives IN the enums.yaml conventions section
- Agent file modifications (T40-02 through T40-07) only add comments and update log format instructions -- no behavioral logic changes
- Agent names in log entries SHALL use lowercase with hyphens: `coder`, `review-coordinator`, `docs-agent`, `orchestrator` (Section 7.6)
- Error handling: if enums.yaml is missing at agent startup, agents SHALL halt with E-010 error

## Risks & Mitigations

- **Risk**: Existing agent files may use different log format variations. **Mitigation**: Search each file for Activity Log patterns before editing. Preserve any additional context the format currently captures.
- **Risk**: Planner may reference "Final" in multiple locations. **Mitigation**: Use grep to find ALL occurrences before patching.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T00:01:00Z - coder - lane=doing - Starting implementation
