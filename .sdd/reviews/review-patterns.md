# Review Patterns

> Last updated: 2026-04-04T18:00:00Z
> Last review: WP01-foundation-scaffolding

Coder: read this file before implementing any WP. These patterns document
mistakes caught in previous reviews. Avoid repeating them.

## Active Patterns

### PAT-001 [spec-adherence] Incomplete agent name reference updates
- **First seen**: WP01 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: When renaming or deprecating an agent, not all referencing agent files are updated to use the new name. Handoff configurations and invocation instructions in other agents still reference the old name.
- **Fix**: When renaming an agent, search ALL agent files for references to the old name and update them. Use grep to verify zero remaining references before marking the task complete.
- **Source**: review-spec SPEC-011

## Resolved

(No resolved patterns yet.)
