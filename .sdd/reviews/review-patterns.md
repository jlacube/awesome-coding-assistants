# Review Patterns

> Last updated: 2026-04-05T18:00:00Z
> Last review: WP18-phase2-api-state-error-skills

Coder: read this file before implementing any WP. These patterns document
mistakes caught in previous reviews. Avoid repeating them.

## Active Patterns

### PAT-006 [spec-adherence] Shared directory misuse for entity deduplication
- **First seen**: WP17 (2026-04-05)
- **Occurrences**: 3
- **Pattern**: Phase 2 skills (plan-data-schemas, plan-interface-contracts, plan-error-catalogs) move entities/interfaces/errors to a shared/ directory when used by 3+ WPs. The spec requires the first WP to define the entity fully and subsequent WPs to import from the first WP's contracts. Section 7.2 reserves shared/ for config-schema only.
- **Fix**: Remove all move-to-shared logic. All deduplication must follow the first-WP-defines pattern regardless of how many WPs share the entity. Only plan-cross-wp-validation may write to shared/ (for config-schema).
- **Source**: review-spec SPEC-016, SPEC-017 (WP17), SPEC-008 (WP18)

### PAT-007 [spec-adherence] Spec literal text deviation in coordinator
- **First seen**: WP15 (2026-04-05)
- **Occurrences**: 2
- **Pattern**: Coordinator implementation substitutes different terms than the spec prescribes. FR-008 specifies "Starter templates or boilerplate repos" but implementation uses "CI/CD best practices". FR-019 specifies "should, appropriate, reasonable" but implementation omits "should" and adds unlisted terms.
- **Fix**: Copy exact terms from spec FRs. Additional terms may be added but spec-prescribed terms must not be omitted.
- **Source**: review-spec SPEC-013, SPEC-025 (WP15)

### PAT-008 [spec-adherence] Implementation guidance template incomplete
- **First seen**: WP16 (2026-04-05)
- **Occurrences**: 1
- **Pattern**: plan-acceptance skill's implementation guidance template omits "error codes" and "validation rules" fields explicitly required by FR-033.3.
- **Fix**: Add "Error handling" and "Spec validation rules" fields to the implementation guidance template, matching the complete list from FR-033.3.
- **Source**: review-spec SPEC-008 (WP16)

## Resolved

### PAT-005 [spec-adherence] Classification level deviation from spec
- **First seen**: WP12 (2026-04-05)
- **Resolved**: WP12 (2026-04-05)
- **Occurrences**: 1
- **Pattern**: Spec prescribes specific enumerated values (e.g., classification levels "public, internal, confidential, restricted") but implementation substitutes a different term ("PII" for "confidential"). Even when the substitution may be arguably better, it deviates from the spec's explicit prescription.
- **Fix**: Use the exact values prescribed by the spec. If a different term is preferable, propose a spec amendment via the "Update Specification" handoff rather than silently deviating.
- **Source**: review-spec SPEC-012

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
