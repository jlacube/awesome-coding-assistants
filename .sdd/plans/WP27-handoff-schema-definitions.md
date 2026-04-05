---
lane: for_review
---

# WP27 - Handoff Schema Definitions

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/006-handoff-schemas-patterns.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | All 8 agent-to-agent handoff schemas exist as validated YAML files in `.github/schemas/` |
| Status | Not Started |
| Independent Test | Run `ls .github/schemas/*.yaml` and verify 8 files; open each and confirm it has `schema: handoff/v1`, `source_agent`, `target_agent`, `required_artifacts`, and `context_fields` |
| Parallelisable | Yes (with WP28) |
| Prompt | `.sdd/plans/WP27-handoff-schema-definitions.md` |

## Objective

Create the `.github/schemas/` directory and author all 8 directional handoff schema files in YAML. Each schema formally defines what the source agent must produce and what the target agent must validate before proceeding. This WP delivers the schema artifacts themselves; agent integration happens in WP29.

## Spec References

FR-001, FR-002, FR-003, FR-006, FR-007, Section 7.1, Section 9.1, Section 9.2

## Tasks

### T27-01 - Create schemas directory and define schema template

- **Description**: Create the `.github/schemas/` directory. Define a reference YAML template that implements the FR-003 structure with `schema: handoff/v1`, `source_agent`, `target_agent`, `description`, `required_artifacts`, `required_state`, `context_fields`, and `validation_rules` top-level keys. This template is used as the starting point for all 8 schema files.
- **Spec refs**: FR-003, FR-007, Section 9.1
- **Parallel**: No (foundation for remaining tasks)
- **Acceptance criteria**:
  - [x] `.github/schemas/` directory exists
  - [x] Template follows the exact YAML structure from FR-003 including the `schema: handoff/v1` version header
  - [x] Every required top-level key per FR-002 is present in the template: `source_agent`, `target_agent`, `required_artifacts`, `context_fields`
  - [x] Optional keys `required_state` and `validation_rules` are included in the template
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Use the exact YAML structure from FR-003 (spec Section 4.1.2) as the reference template
  - The `schema` field value SHALL be the string `handoff/v1` per FR-007
  - YAML 1.2 compliant: https://yaml.org/spec/1.2.2/
  - Known pitfall: YAML strings with special characters need quoting (e.g., agent names like "2. Spec Architect")
  - Files to create: `.github/schemas/` directory (scaffold)

### T27-02 - Author ideation-to-spec.schema.yaml

- **Description**: Author the schema defining the handoff from Brainstorming/Ideation agent to Spec Architect. Required artifacts include the idea brief file. Context fields include `brief_path`.
- **Spec refs**: FR-001 (item 1), FR-002
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schema file exists at `.github/schemas/ideation-to-spec.schema.yaml`
  - [x] `source_agent` is set to the Ideation/Brainstorming agent name
  - [x] `target_agent` is set to the Spec Architect agent name
  - [x] `required_artifacts` list includes the idea brief file path with `exists: true` validation
  - [x] `context_fields` includes `brief_path` as required string
  - [x] All 6 required fields from FR-002 are present
- **Test requirements**: none
- **Depends on**: T27-01
- **Implementation Guidance**:
  - Source agent name: check `.github/agents/brainstorming.agent.md` and `.github/agents/ideation.agent.md` YAML frontmatter for exact agent name
  - Target agent: "2. Spec Architect" (from spec-architect.agent.md)
  - Required artifacts: `.sdd/ideas/{NNN}-*.md` (idea brief file)
  - Context fields: `brief_path` (string, required)
  - Error handling: If brief file does not exist, validation fails with "Idea brief does not exist at {brief_path}"

### T27-03 - Author spec-to-planner.schema.yaml

- **Description**: Author the schema defining the handoff from Spec Architect to Planner. This is the reference schema (FR-003 provides its full example). Required artifacts include the validated spec file and companion artifacts directory.
- **Spec refs**: FR-001 (item 2), FR-002, FR-003
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schema file exists at `.github/schemas/spec-to-planner.schema.yaml`
  - [x] Content matches the reference example from FR-003 in structure and intent
  - [x] `required_artifacts` includes spec file with `Status: Validated` field validation
  - [x] `required_artifacts` includes companion artifacts directory with `min_files: 1`
  - [x] `context_fields` includes `spec_path` and `artifacts_dir` as required strings
  - [x] `validation_rules` includes `file_exists` and `field_value` checks
  - [x] `required_state` includes `spec.status == 'Validated'` condition
- **Test requirements**: BDD (Scenario: Valid handoff passes schema; Scenario: Invalid handoff fails schema)
- **Depends on**: T27-01
- **Implementation Guidance**:
  - Use the exact example from FR-003 (spec Section 4.1.2) as the base
  - Source agent: "2. Spec Architect"; Target agent: "3. Planner"
  - Required artifacts: `.sdd/specs/{NNN}-{name}.spec.md` with Status=Validated, `.sdd/specs/artifacts/{NNN}-{name}/` with min_files=1
  - Error: "Spec must be Validated before planning" (from FR-003 required_state)
  - Error: "Spec file does not exist at {spec_path}" (from FR-003 validation_rules)
  - Error: "Spec status is not Validated" (from FR-003 validation_rules)

### T27-04 - Author planner-to-coder.schema.yaml

- **Description**: Author the schema defining the handoff from Planner to Coder. Required artifacts include the plan directory with WP files and optionally contract files.
- **Spec refs**: FR-001 (item 3), FR-002
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schema file exists at `.github/schemas/planner-to-coder.schema.yaml`
  - [x] `source_agent` is "3. Planner" and `target_agent` is "4. Coder"
  - [x] `required_artifacts` includes plan WP file with `exists: true`
  - [x] `context_fields` includes `wp_path`, `spec_path`, `contracts_dir` as fields
  - [x] `validation_rules` check WP file existence
  - [x] All 6 required fields from FR-002 are present
- **Test requirements**: none
- **Depends on**: T27-01
- **Implementation Guidance**:
  - Source: "3. Planner"; Target: "4. Coder"
  - Required artifacts: `.sdd/plans/WP<NN>-<slug>.md` (the specific WP to implement)
  - Context fields: `wp_path` (required), `spec_path` (required), `contracts_dir` (required)
  - Required state: WP lane should be `planned` or similar
  - Reference existing coder.agent.md for expected input format

### T27-05 - Author coder-to-reviewer.schema.yaml

- **Description**: Author the schema defining the handoff from Coder to Review Coordinator. Required artifacts include the WP file with implementation and the implementation files themselves.
- **Spec refs**: FR-001 (item 4), FR-002
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schema file exists at `.github/schemas/coder-to-reviewer.schema.yaml`
  - [x] `source_agent` is "4. Coder" and `target_agent` is "5. Review Coordinator"
  - [x] `required_artifacts` includes implementation files with validation
  - [x] `context_fields` includes `wp_path`, `spec_path`, and implementation-related fields
  - [x] All 6 required fields from FR-002 are present
- **Test requirements**: BDD (Scenario: Invalid handoff fails schema - missing implementation)
- **Depends on**: T27-01
- **Implementation Guidance**:
  - Source: "4. Coder"; Target: "5. Reviewer"
  - Required artifacts: WP file (with lane=for_review or equivalent), implementation files
  - Context fields: `wp_path`, `spec_path`, `contracts_dir`
  - Required state: WP lane should indicate implementation is complete (e.g., `for_review`)
  - Reference existing review-coordinator.agent.md for expected input format

### T27-06 - Author rework handoff schemas

- **Description**: Author 3 rework/feedback loop schemas: reviewer-to-coder.schema.yaml (review findings for rework), reviewer-to-spec.schema.yaml (spec gaps found during review), and planner-to-spec.schema.yaml (auto-loop for spec completeness gaps).
- **Spec refs**: FR-001 (items 5-7), FR-002
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] `.github/schemas/reviewer-to-coder.schema.yaml` exists with Review Coordinator as source and Coder as target
  - [x] `.github/schemas/reviewer-to-spec.schema.yaml` exists with Review Coordinator as source and Spec Architect as target
  - [x] `.github/schemas/planner-to-spec.schema.yaml` exists with Planner as source and Spec Architect as target
  - [x] Each schema includes review findings or gap report as required artifacts
  - [x] Each schema has context_fields relevant to the rework scenario
  - [x] All 3 schemas have all 6 required fields from FR-002
- **Test requirements**: none
- **Depends on**: T27-01
- **Implementation Guidance**:
  - reviewer-to-coder: Source "5. Reviewer", Target "4. Coder"; required artifacts include review report with findings; context: `wp_path`, `review_report_path`
  - reviewer-to-spec: Source "5. Reviewer", Target "2. Spec Architect"; required artifacts include review report with spec gaps; context: `spec_path`, `gap_report`
  - planner-to-spec: Source "3. Planner", Target "2. Spec Architect"; required artifacts include gap report from completeness pre-check; context: `spec_path`, `gap_report`
  - Reference existing agent handoff patterns in YAML frontmatter `handoffs:` declarations

### T27-07 - Author orchestrator-handoff.schema.yaml

- **Description**: Author the generic schema for Orchestrator-to-any-agent handoffs. This schema defines the common contract the Orchestrator uses when delegating to any pipeline agent.
- **Spec refs**: FR-001 (item 8), FR-002
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Schema file exists at `.github/schemas/orchestrator-handoff.schema.yaml`
  - [x] `source_agent` is the Orchestrator agent name
  - [x] `target_agent` uses a wildcard or generic indicator for "any agent"
  - [x] `context_fields` include at minimum the target agent identifier and user request
  - [x] All 6 required fields from FR-002 are present
- **Test requirements**: none
- **Depends on**: T27-01
- **Implementation Guidance**:
  - Source: "1. Orchestrator" (check orchestrator.agent.md for exact name)
  - Target: generic -- this schema defines the minimum handoff contract for any delegation
  - Context fields: `target_agent`, `user_request`, and any common fields all agents expect
  - Reference existing orchestrator.agent.md for current handoff patterns

### T27-08 - Verify schema compliance and add maintenance documentation

- **Description**: Verify all 8 schema files parse as valid YAML, conform to the `handoff/v1` version, and contain all required top-level keys. Add a brief maintenance note to each schema about update process (FR-006).
- **Spec refs**: FR-006, FR-007
- **Parallel**: No (verification step)
- **Acceptance criteria**:
  - [x] All 8 schema files parse as valid YAML 1.2
  - [x] All 8 schema files have `schema: handoff/v1` header
  - [x] All 8 schemas have all required top-level keys: `schema`, `source_agent`, `target_agent`, `required_artifacts`, `context_fields`
  - [x] Each schema includes a comment or description noting that schema changes SHALL be committed with agent changes (FR-006)
  - [x] No schema file contains executable code or template expressions (NFR-003)
- **Test requirements**: none
- **Depends on**: T27-02, T27-03, T27-04, T27-05, T27-06, T27-07
- **Implementation Guidance**:
  - Validate YAML parsing: open each file and confirm no parse errors
  - Check for NFR-003 compliance: no executable expressions, no template strings, purely declarative
  - FR-006: Add a comment in each schema file noting that when an agent's interface changes the corresponding schema must be updated and committed together
  - FR-007: If a future incompatible change is needed, the version field increments to `handoff/v2`

## Implementation Notes

All deliverables are YAML files -- no executable code, no build system, no test framework. "Testing" means opening each YAML file and verifying it parses correctly with the required structure.

The spec-to-planner schema (T27-03) should be written first as it is the reference example provided in the spec (FR-003). Other schemas follow the same structure adapted to their specific handoff.

Schema files should reference actual artifact paths used by each agent. Check existing agent `.agent.md` files for current handoff patterns to ensure schemas accurately reflect agent interfaces.

## Parallel Opportunities

T27-02 through T27-07 can all be worked in parallel after T27-01 completes. T27-08 is the final verification step.

## Risks & Mitigations

- **Risk**: Agent names in schemas may not match exact frontmatter names. **Mitigation**: Cross-reference each `.github/agents/*.agent.md` file's YAML frontmatter `name:` field.
- **Risk**: Required artifacts paths use glob patterns that may not cover all edge cases. **Mitigation**: Use the same glob patterns already used in agent files; add validation_rules for specifics.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T00:00:00Z - coder - T27-01 - completed - Created .github/schemas/ directory with spec-to-planner as reference template
- 2026-04-06T00:00:00Z - coder - T27-02 - completed - Authored ideation-to-spec.schema.yaml
- 2026-04-06T00:00:00Z - coder - T27-03 - completed - Authored spec-to-planner.schema.yaml (reference schema per FR-003)
- 2026-04-06T00:00:00Z - coder - T27-04 - completed - Authored planner-to-coder.schema.yaml
- 2026-04-06T00:00:00Z - coder - T27-05 - completed - Authored coder-to-reviewer.schema.yaml
- 2026-04-06T00:00:00Z - coder - T27-06 - completed - Authored reviewer-to-coder, reviewer-to-spec, planner-to-spec schemas
- 2026-04-06T00:00:00Z - coder - T27-07 - completed - Authored orchestrator-handoff.schema.yaml
- 2026-04-06T00:00:00Z - coder - T27-08 - completed - All 8 schemas validated: YAML parse OK, required keys present, handoff/v1 version, FR-006 comments, NFR-003 clean
- 2026-04-06T00:00:00Z - coder - lane=for_review - All tasks complete, tests passing, coverage met
