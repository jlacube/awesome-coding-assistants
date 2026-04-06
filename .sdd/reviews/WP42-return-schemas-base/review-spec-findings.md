---
skill: review-spec
wp: WP42-return-schemas-base
date: 2026-04-07T00:10:00Z
finding_counts:
  pass: 13
  warn: 1
  fail: 0
  na: 0
files_reviewed:
  - .github/schemas/base-handoff.schema.yaml
  - .github/schemas/reviewer-to-orchestrator.schema.yaml
  - .github/schemas/coder-complete-to-orchestrator.schema.yaml
  - .github/schemas/docs-agent-to-orchestrator.schema.yaml
  - .github/schemas/spec-to-planner.schema.yaml
  - .github/schemas/coder-to-reviewer.schema.yaml
status: WARN
---

# review-spec Findings for WP42

## In-Scope FRs

FR-021, FR-022, FR-023, FR-024, FR-025, FR-026, FR-027, FR-049, FR-050, FR-051

## In-Scope Success Criteria

SC-003, SC-010

---

### SPEC-001 [PASS]

**FR-021**: Schema file SHALL exist at `.github/schemas/reviewer-to-orchestrator.schema.yaml`.

File exists. Follows handoff/v1 format.

---

### SPEC-002 [PASS]

**FR-022**: reviewer-to-orchestrator schema SHALL require context fields: wp_path (string, required), verdict (enum: approved/changes_required, required), updated_lane (enum from lane values, required).

All three fields present with correct types, required=true, and enum/enum_ref constraints.
- `wp_path`: type string, required true
- `verdict`: type string, required true, enum: ["approved", "changes_required"]
- `updated_lane`: type string, required true, enum_ref: ".github/schemas/enums.yaml#lane"

Validation rules include `verdict_lane_consistency` check per Section 7.7: "approved->done, changes_required->to_do".

---

### SPEC-003 [PASS]

**FR-023**: Schema file SHALL exist at `.github/schemas/coder-complete-to-orchestrator.schema.yaml`.

File exists. Follows handoff/v1 format.

---

### SPEC-004 [PASS]

**FR-024**: coder-complete-to-orchestrator SHALL require: wp_path (string, required) and lane_confirmation (literal "for_review", required).

Both fields present. `lane_confirmation` has `literal: "for_review"` and validation rule with `check: "literal_value"` and `expected: "for_review"`. Error message matches spec.

---

### SPEC-005 [PASS]

**FR-025**: Schema file SHALL exist at `.github/schemas/docs-agent-to-orchestrator.schema.yaml`.

File exists. Follows handoff/v1 format.

---

### SPEC-006 [PASS]

**FR-026**: docs-agent-to-orchestrator SHALL require: wp_path (string, required) and docs_completed (boolean, required).

Both fields present. `docs_completed` has type boolean, required true. Validation rule checks `literal_value` expected true. Error message: "docs_completed must be true. False is not a valid completion signal." matches spec.

---

### SPEC-007 [PASS]

**FR-027**: All three return schemas SHALL follow handoff/v1 format with schema, source_agent, target_agent, description, required_artifacts, required_state, context_fields, validation_rules sections.

All three schemas verified:
- reviewer-to-orchestrator: all 8 required sections present
- coder-complete-to-orchestrator: all 8 required sections present
- docs-agent-to-orchestrator: all 8 required sections present

---

### SPEC-008 [PASS]

**FR-049**: Shared base schema SHALL exist at `.github/schemas/base-handoff.schema.yaml`. Individual schemas SHALL fall back to inline rules if base is missing.

File exists with `schema: base/v1`. All individual schemas contain inline validation_rules with fallback comments: "# Inline rules (fallback if base schema is unavailable)".

---

### SPEC-009 [PASS]

**FR-050**: Base schema SHALL define validation patterns: wp_file_exists, lane_value_valid, file_path_format.

All three patterns present with required fields:
- `wp_file_exists`: check + error fields
- `lane_value_valid`: check + enum_ref + error fields. enum_ref correctly points to `.github/schemas/enums.yaml#lane`
- `file_path_format`: check + pattern + error fields. Pattern `"^\\.sdd/plans/WP\\d{2}-[a-z0-9-]+\\.md$"` matches spec Section 10.2.4

---

### SPEC-010 [PASS]

**FR-051**: Individual schemas SHALL reference base using `base_schema` field. At least 2 individual schemas.

Five schemas reference the base:
1. reviewer-to-orchestrator.schema.yaml
2. coder-complete-to-orchestrator.schema.yaml
3. docs-agent-to-orchestrator.schema.yaml
4. spec-to-planner.schema.yaml (updated in T42-05)
5. coder-to-reviewer.schema.yaml (updated in T42-05)

Exceeds the minimum of 2.

---

### SPEC-011 [PASS]

**SC-003**: Every agent-to-agent handoff has a corresponding schema file.

All 3 return handoff schemas now exist, completing the handoff coverage.

---

### SPEC-012 [PASS]

**SC-010**: Common validation patterns extracted into shared base schema referenced by individual schemas.

base-handoff.schema.yaml exists with 3 patterns. Referenced by 5 individual schemas.

---

### SPEC-013 [PASS]

**Section 7.3 data model**: Handoff schema structure compliance.

All new schemas match the Section 7.3 data model:
- schema: handoff/v1 (matches `handoff/v\d+` regex)
- source_agent/target_agent: present
- base_schema: present (optional field used)
- required_artifacts, required_state, context_fields, validation_rules: all present
- version_history: absent (optional, deferred to WP46)

---

### SPEC-014 [WARN]

**T42-06 / Section 7.3**: Agent name consistency across schemas.

`reviewer-to-orchestrator.schema.yaml` uses `source_agent: "5. Reviewer"` but all other schemas referencing this agent use `"5. Review Coordinator"`:
- `coder-to-reviewer.schema.yaml`: `target_agent: "5. Review Coordinator"`
- `reviewer-to-coder.schema.yaml`: `source_agent: "5. Review Coordinator"`
- `reviewer-to-spec.schema.yaml`: `source_agent: "5. Review Coordinator"`

The agent file is `review-coordinator.agent.md`, which matches "Review Coordinator" not "Reviewer".

File: `.github/schemas/reviewer-to-orchestrator.schema.yaml#L11`
Expected: `source_agent: "5. Review Coordinator"`

Note: The WP plan's T42-02 acceptance criteria specified "5. Reviewer", so the Coder followed the plan. The inconsistency originates in the WP plan.
