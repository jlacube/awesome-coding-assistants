# Review Patterns

> Last updated: 2026-04-05T01:00:00Z
> Last review: WP06-p2-skills

Coder: read this file before implementing any WP. These patterns document
mistakes caught in previous reviews. Avoid repeating them.

## Active Patterns

(none)

## Resolved

### PAT-004 [spec-adherence] Incomplete OWASP checklist item coverage
- **First seen**: WP04 (2026-04-04)
- **Resolved**: WP04 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: When implementing a checklist from the spec, not all enumerated items are included. The spec's FR-034 lists specific items per OWASP category, but 4 of ~68 items were omitted from the implementation.
- **Fix**: Cross-reference each category in the implementation against FR-034's category definitions line by line. Count items in both to verify none are missing.
- **Source**: review-spec SPEC-006, review-security SEC-017/SEC-018/SEC-019/SEC-020

## Resolved

### PAT-002 [spec-adherence] Spec SHALL deviation on optional handling
- **First seen**: WP02 (2026-04-04)
- **Resolved**: WP02 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: Implementation treats a chain item as optional ("record a note but continue") when the spec uses SHALL language requiring halt on ANY missing item. Deviates from strict spec language without proposing a spec amendment.
- **Fix**: Follow spec SHALL obligations exactly. If the obligation is overly strict, propose a spec amendment via the "Update Specification" handoff rather than silently deviating.
- **Source**: review-spec SPEC-002

### PAT-003 [docs] Missing project documentation directory
- **First seen**: WP02 (2026-04-04)
- **Resolved**: WP02 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: `.sdd/docs/` directory and standard documentation files (architecture.md, user-guide.md, developer-guide.md) are missing. No WP in the plan is assigned to create them.
- **Fix**: Create a documentation WP or add documentation tasks to existing WPs. At minimum, create architecture.md, user-guide.md, and developer-guide.md for user-facing and developer-facing components.
- **Source**: review-docs DOC-001, DOC-005, DOC-006, DOC-009, DOC-010

### PAT-001 [spec-adherence] Incomplete agent name reference updates
- **First seen**: WP01 (2026-04-04)
- **Resolved**: WP01 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: When renaming or deprecating an agent, not all referencing agent files are updated to use the new name. Handoff configurations and invocation instructions in other agents still reference the old name.
- **Fix**: When renaming an agent, search ALL agent files for references to the old name and update them. Use grep to verify zero remaining references before marking the task complete.
- **Source**: review-spec SPEC-011
