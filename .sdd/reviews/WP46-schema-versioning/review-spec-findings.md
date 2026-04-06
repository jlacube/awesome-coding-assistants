---
skill: review-spec
wp: WP46-schema-versioning
date: "2026-04-07T00:10:00Z"
status: PASS
files_reviewed:
  - .github/schemas/ideation-to-spec.schema.yaml
  - .github/schemas/spec-to-planner.schema.yaml
  - .github/schemas/planner-to-coder.schema.yaml
  - .github/schemas/coder-to-reviewer.schema.yaml
  - .github/schemas/reviewer-to-coder.schema.yaml
  - .github/schemas/reviewer-to-spec.schema.yaml
  - .github/schemas/planner-to-spec.schema.yaml
  - .github/schemas/orchestrator-handoff.schema.yaml
  - .github/schemas/reviewer-to-orchestrator.schema.yaml
  - .github/schemas/coder-complete-to-orchestrator.schema.yaml
  - .github/schemas/docs-agent-to-orchestrator.schema.yaml
  - .github/schemas/base-handoff.schema.yaml
  - .sdd/docs/developer-guide.md
finding_counts:
  pass: 6
  warn: 0
  fail: 0
  na: 0
---

# review-spec Findings -- WP46-schema-versioning

## Checklist

### SPEC-001 [PASS] FR-044: version_history in all schema files
All 12 schema files in `.github/schemas/` contain a `version_history` YAML array with at least one entry.

### SPEC-002 [PASS] FR-045: version_history entry structure
Every version_history entry contains the required three fields: `version` (string), `date` (ISO-8601), `description` (string). Base schema correctly uses "base/v1" instead of "handoff/v1".

### SPEC-003 [PASS] FR-046: Breaking changes require version increment
Developer guide "When to Increment the Version" section documents breaking changes (removing fields, changing types, removing enum values, renaming fields) with concrete example. `.sdd/docs/developer-guide.md#L214-L222`.

### SPEC-004 [PASS] FR-047: Additive changes retain version
Developer guide "When to Keep the Version" section documents additive changes (new optional fields, new enum values, new optional rules) with concrete example. `.sdd/docs/developer-guide.md#L224-L232`.

### SPEC-005 [PASS] FR-048: Placeholder regex patterns
10 schemas with path placeholders have `placeholder_patterns` sections. All four spec-defined placeholders are correctly mapped: `{NNN}` -> `\d{2,3}`, `{name}` -> `[a-z0-9-]+`, `{slug}` -> `[a-z0-9-]+`, `{NN}` -> `\d{2}`. Two schemas without artifact path placeholders (orchestrator-handoff, base-handoff) correctly omit the section.

### SPEC-006 [PASS] SC-009: Success criteria met
Schema files contain version_history metadata and the versioning protocol is documented in the developer guide.
