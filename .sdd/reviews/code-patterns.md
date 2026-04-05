# Code Patterns

## Active Patterns

(none)

## Retired Patterns

### PAT-CODE-001: Incomplete agent name reference updates
- **Status**: retired
- **Added**: 2026-04-04
- **Source**: Review of WP01 -- review-spec SPEC-011
- **Trigger**: When renaming or deprecating an agent, handoff configurations and invocation instructions in other agents still reference the old name
- **Prevention**: When renaming an agent, search ALL agent files for references to the old name and update them. Use grep to verify zero remaining references before marking the task complete.

### PAT-CODE-002: Spec SHALL deviation on optional handling
- **Status**: retired
- **Added**: 2026-04-04
- **Source**: Review of WP02 -- review-spec SPEC-002
- **Trigger**: Implementation treats a chain item as optional when the spec uses SHALL language requiring halt on ANY missing item
- **Prevention**: Follow spec SHALL obligations exactly. If the obligation is overly strict, propose a spec amendment via the "Update Specification" handoff rather than silently deviating.

### PAT-CODE-003: Incomplete OWASP checklist item coverage
- **Status**: retired
- **Added**: 2026-04-04
- **Source**: Review of WP04 -- review-spec SPEC-006, review-security SEC-017/SEC-018/SEC-019/SEC-020
- **Trigger**: When implementing a checklist from the spec, not all enumerated items are included
- **Prevention**: Cross-reference each category in the implementation against the spec's category definitions line by line. Count items in both to verify none are missing.

### PAT-CODE-004: Classification level deviation from spec
- **Status**: retired
- **Added**: 2026-04-05
- **Source**: Review of WP12 -- review-spec SPEC-012
- **Trigger**: Spec prescribes specific enumerated values but implementation substitutes a different term
- **Prevention**: Use the exact values prescribed by the spec. If a different term is preferable, propose a spec amendment via the "Update Specification" handoff rather than silently deviating.

### PAT-CODE-005: Shared directory misuse for entity deduplication
- **Status**: retired
- **Added**: 2026-04-05
- **Source**: Review of WP17 -- review-spec SPEC-016, SPEC-017; Review of WP18 -- SPEC-008
- **Trigger**: Phase 2 skills move entities/interfaces/errors to a shared/ directory when used by 3+ WPs instead of following the first-WP-defines pattern
- **Prevention**: Remove all move-to-shared logic. All deduplication must follow the first-WP-defines pattern regardless of how many WPs share the entity. Only plan-cross-wp-validation may write to shared/ (for config-schema).

### PAT-CODE-006: Spec literal text deviation in coordinator
- **Status**: retired
- **Added**: 2026-04-05
- **Source**: Review of WP15 -- review-spec SPEC-013, SPEC-025
- **Trigger**: Coordinator implementation substitutes different terms than the spec prescribes for research topics or soft-obligation keywords
- **Prevention**: Copy exact terms from spec FRs. Additional terms may be added but spec-prescribed terms must not be omitted.

### PAT-CODE-007: Implementation guidance template incomplete
- **Status**: retired
- **Added**: 2026-04-05
- **Source**: Review of WP16 -- review-spec SPEC-008
- **Trigger**: Skill implementation guidance template omits fields explicitly required by the spec FR
- **Prevention**: Add all fields listed in the spec FR to the implementation guidance template. Cross-check the template against the FR's complete list of required fields.
