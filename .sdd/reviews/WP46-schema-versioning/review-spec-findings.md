---
skill: review-spec
wp: WP46-schema-versioning
spec: .sdd/specs/010-sdd-pipeline-hardening.spec.md
reviewed_at: 2026-04-07T12:00:00Z
status: completed
finding_counts:
  pass: 6
  warn: 0
  fail: 0
  na: 0
files_reviewed:
  - .github/schemas/base-handoff.schema.yaml
  - .github/schemas/coder-complete-to-orchestrator.schema.yaml
  - .github/schemas/coder-to-reviewer.schema.yaml
  - .github/schemas/docs-agent-to-orchestrator.schema.yaml
  - .github/schemas/ideation-to-spec.schema.yaml
  - .github/schemas/orchestrator-handoff.schema.yaml
  - .github/schemas/planner-to-coder.schema.yaml
  - .github/schemas/planner-to-spec.schema.yaml
  - .github/schemas/reviewer-to-coder.schema.yaml
  - .github/schemas/reviewer-to-orchestrator.schema.yaml
  - .github/schemas/reviewer-to-spec.schema.yaml
  - .github/schemas/spec-to-planner.schema.yaml
  - .sdd/docs/developer-guide.md
---

# review-spec Findings for WP46-schema-versioning (Re-review Round 2)

## Findings

### SPEC-001 [PASS]
**FR-044**: Each handoff schema file SHALL include a version_history section.
All 12 schema files in .github/schemas/ contain a version_history YAML array. Verified via grep search confirming matches in every file.

### SPEC-002 [PASS]
**FR-045**: A version history entry SHALL contain: version, date, description.
All 12 schema files verified. Every version_history entry contains all three required fields: version (string), date (ISO-8601), description (string). Base schema uses "base/v1" per T46-02; all others use "handoff/v1".

### SPEC-003 [PASS]
**FR-046**: Breaking changes SHALL increment the schema version.
Developer guide at .sdd/docs/developer-guide.md#L191 documents breaking change rules matching spec exactly.

### SPEC-004 [PASS]
**FR-047**: Additive changes SHALL retain the current schema version number.
Developer guide documents additive change rules. Both breaking and additive changes require a version_history entry. Matches spec.

### SPEC-005 [PASS]
**FR-048**: Each schema SHALL validate artifact path placeholders using defined regex patterns.
10 schemas with path placeholders have placeholder_patterns sections. 2 schemas without path placeholders (base-handoff, orchestrator-handoff) correctly omit the section. All 4 spec-defined patterns verified correct.

Note: planner-to-coder.schema.yaml uses {WP-slug} in contracts path but this placeholder is not in FR-048's defined set. This is a spec gap, not an implementation defect.

### SPEC-006 [PASS]
**SC-009**: Schema versioning protocol documented and schema files contain version_history sections.
Developer guide contains versioning protocol section; all 12 schema files contain version_history entries. SC-009 fully satisfied.
