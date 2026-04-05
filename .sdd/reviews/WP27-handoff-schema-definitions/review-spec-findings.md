---
skill: review-spec
wp: WP27-handoff-schema-definitions
spec: .sdd/specs/006-handoff-schemas-patterns.spec.md
reviewed_at: 2026-04-06T14:00:00Z
status: completed
finding_counts:
  pass: 7
  warn: 1
  fail: 0
  na: 1
files_reviewed:
  - .github/schemas/ideation-to-spec.schema.yaml
  - .github/schemas/spec-to-planner.schema.yaml
  - .github/schemas/planner-to-coder.schema.yaml
  - .github/schemas/coder-to-reviewer.schema.yaml
  - .github/schemas/reviewer-to-coder.schema.yaml
  - .github/schemas/reviewer-to-spec.schema.yaml
  - .github/schemas/planner-to-spec.schema.yaml
  - .github/schemas/orchestrator-handoff.schema.yaml
---

# review-spec Findings for WP27-handoff-schema-definitions

## Summary

WP27 references FR-001, FR-002, FR-003, FR-006, FR-007 from spec 006. All 8 schema files exist in the correct directory. All schemas comply with the required structure, use correct agent names (verified against `.github/agents/*.agent.md` frontmatter), and include maintenance notes per FR-006. The spec-to-planner schema matches the FR-003 reference example. One minor deviation: the orchestrator schema has empty `required_artifacts` which conflicts with the data model constraint of `min 1` (Section 7.1), but this is pragmatically correct for a generic delegation schema.

Total FRs evaluated: 5 (FR-001, FR-002, FR-003, FR-006, FR-007). All Compliant. 1 SC evaluated, 1 N/A.

## Findings

### SPEC-001 [PASS]
- **FR**: FR-001 - Schema file enumeration
- **Classification**: Compliant
- **Evidence**: All 8 schema files exist at the paths specified in FR-001:
  1. `.github/schemas/ideation-to-spec.schema.yaml`
  2. `.github/schemas/spec-to-planner.schema.yaml`
  3. `.github/schemas/planner-to-coder.schema.yaml`
  4. `.github/schemas/coder-to-reviewer.schema.yaml`
  5. `.github/schemas/reviewer-to-coder.schema.yaml`
  6. `.github/schemas/reviewer-to-spec.schema.yaml`
  7. `.github/schemas/planner-to-spec.schema.yaml`
  8. `.github/schemas/orchestrator-handoff.schema.yaml`

### SPEC-002 [PASS]
- **FR**: FR-002 - Required schema fields
- **Classification**: Compliant
- **Evidence**: All 8 schemas contain all 6 required fields: `source_agent`, `target_agent`, `required_artifacts`, `required_state`, `context_fields`, `validation_rules`. Verified by reading each file.

### SPEC-003 [PASS]
- **FR**: FR-003 - Schema format (reference example)
- **Classification**: Compliant
- **Evidence**: `spec-to-planner.schema.yaml` matches the FR-003 reference YAML structure exactly: same top-level keys, same artifact paths, same validation rules, same context fields, same required_state conditions and error messages.

### SPEC-004 [PASS]
- **FR**: FR-006 - Schema maintenance comments
- **Classification**: Compliant
- **Evidence**: All 8 schemas include FR-006 maintenance comment: "When the [agent] interface changes, update this schema and commit it together with the agent changes." All also include FR-007 version comment.

### SPEC-005 [PASS]
- **FR**: FR-007 - Schema versioning
- **Classification**: Compliant
- **Evidence**: All 8 schemas use `schema: handoff/v1` as the first YAML key.

### SPEC-006 [PASS]
- **FR**: Agent name correctness (cross-reference)
- **Classification**: Compliant
- **Evidence**: Verified all agent names against `.github/agents/*.agent.md` frontmatter `name:` fields:
  - "0. Orchestrator" matches orchestrator.agent.md
  - "1. Ideation" matches ideation.agent.md
  - "2. Spec Architect" matches spec-architect.agent.md
  - "3. Planner" matches planner.agent.md
  - "4. Coder" matches coder.agent.md
  - "5. Review Coordinator" matches review-coordinator.agent.md

### SPEC-007 [PASS]
- **FR**: FR-003 required top-level keys
- **Classification**: Compliant
- **Evidence**: All 8 schemas have the 5 required top-level keys from FR-003: `schema`, `source_agent`, `target_agent`, `required_artifacts`, `context_fields`. All also have the optional `description`, `required_state`, and `validation_rules` keys.

### SPEC-008 [WARN]
- **FR**: Section 7.1 Data Model - required_artifacts constraint
- **Classification**: Minor deviation
- **File**: .github/schemas/orchestrator-handoff.schema.yaml#L14
- **Description**: The data model (Section 7.1) specifies `required_artifacts` has constraint `min 1`. The orchestrator schema has `required_artifacts: []` (empty list). This is pragmatically correct -- the Orchestrator delegates user requests without producing artifacts -- but technically violates the data model constraint.
- **Evidence**: Section 7.1 table row: `required_artifacts | list(ArtifactSpec) | min 1 | Files that must exist`. Implementation: `required_artifacts: []`.

### SPEC-009 [N/A]
- **SC**: SC-002 - Schema validation behavior
- **Justification**: SC-002 tests validation behavior at handoff time. This is runtime behavior implemented in WP29 (agent integration), not in WP27 (schema definition). Deferred verification.
