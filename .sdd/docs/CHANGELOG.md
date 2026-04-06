# Changelog

> This document is maintained by the `doc-changelog` skill. Entries are ordered newest first.

---

## [WP42] - Return Handoff Schemas & Shared Base (2026-04-07)

### Changes

- Created shared base handoff schema at `.github/schemas/base-handoff.schema.yaml` defining reusable validation patterns (wp_file_exists, lane_value_valid, file_path_format) for use by individual handoff schemas
- Created return handoff schema `reviewer-to-orchestrator.schema.yaml` formalizing the Review Coordinator's completion signal to the Orchestrator with `wp_path`, `verdict`, and `updated_lane` context fields
- Created return handoff schema `coder-complete-to-orchestrator.schema.yaml` formalizing the Coder's completion signal to the Orchestrator with `wp_path` and `lane_confirmation` context fields
- Created return handoff schema `docs-agent-to-orchestrator.schema.yaml` formalizing the Docs Agent's completion signal to the Orchestrator with `wp_path` and `docs_completed` context fields
- Linked existing schemas (`spec-to-planner.schema.yaml`, `coder-to-reviewer.schema.yaml`) to the base schema via `base_schema` field, replacing duplicated validation rules with shared references

### Breaking Changes

None.

## [WP41] - WP Frontmatter Extensions (2026-04-07)

### Changes

- Added optional `review_cycles` (integer, default 0) and `docs_completed` (boolean, default false) frontmatter fields to WP files, replacing Activity Log text parsing with structured YAML state
- Updated the Review Coordinator to increment `review_cycles` by 1 each time it sets a WP's lane to `to_do` (rework requested)
- Updated the Docs Agent to set `docs_completed: true` in WP frontmatter upon successful documentation generation
- Updated the Orchestrator to read `review_cycles` from frontmatter for escalation decisions (`review_cycles >= 3`) instead of scanning Activity Log entries
- Updated the Orchestrator to read `docs_completed` from frontmatter to determine whether to invoke the Docs Agent, instead of scanning Activity Log entries
- All agents treat absent fields as their defaults (0 and false), maintaining backward compatibility with existing WP files

### Breaking Changes

None.

## [WP40] - Enum Registry & Canonical Conventions (2026-04-07)

### Changes

- Created central enum registry at `.github/schemas/enums.yaml` defining all pipeline-wide enumeration values (`lane`, `spec_status`, `pipeline_stage`, `review_status`) as a single source of truth
- Added canonical Activity Log format (`<ISO-8601-timestamp> - <agent-name> - <action> - <details>`) stored in the registry's `conventions` section
- Added `<!-- Enum source: .github/schemas/enums.yaml -->` reference comments to all 6 agent files (Orchestrator, Coder, Review Coordinator, Docs Agent, Planner, Spec Architect)
- Updated Coder, Review Coordinator, and Docs Agent Activity Log Protocol sections to use the canonical format
- Removed deprecated "Final" spec status value from Planner agent instructions

### Breaking Changes

- **Planner spec_status**: "Final" is no longer a valid spec status value. Valid values are now Draft, Validated, Approved only. Migration: replace any "Final" references with "Approved".
