# Changelog

> This document is maintained by the `doc-changelog` skill. Entries are ordered newest first.

---

## [WP40] - Enum Registry & Canonical Conventions (2026-04-07)

### Changes

- Created central enum registry at `.github/schemas/enums.yaml` defining all pipeline-wide enumeration values (`lane`, `spec_status`, `pipeline_stage`, `review_status`) as a single source of truth
- Added canonical Activity Log format (`<ISO-8601-timestamp> - <agent-name> - <action> - <details>`) stored in the registry's `conventions` section
- Added `<!-- Enum source: .github/schemas/enums.yaml -->` reference comments to all 6 agent files (Orchestrator, Coder, Review Coordinator, Docs Agent, Planner, Spec Architect)
- Updated Coder, Review Coordinator, and Docs Agent Activity Log Protocol sections to use the canonical format
- Removed deprecated "Final" spec status value from Planner agent instructions

### Breaking Changes

- **Planner spec_status**: "Final" is no longer a valid spec status value. Valid values are now Draft, Validated, Approved only. Migration: replace any "Final" references with "Approved".
