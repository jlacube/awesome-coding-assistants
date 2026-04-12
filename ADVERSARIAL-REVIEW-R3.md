# Adversarial Review — Round 3

**Date**: 2025-07-10  
**Scope**: `.github/agents/`, `.github/skills/`, `.github/schemas/`, `.github/prompts/`  
**Reviewer**: Automated adversarial audit  
**Baseline**: ADVERSARIAL-REVIEW-R2.md (all R2 issues verified fixed — see Appendix A)

---

## Executive Summary

All 15 issues from Round 2 have been verified as fixed. This round discovered **2 significant**, **5 moderate**, and **6 minor** new issues. No critical issues remain. The most impactful findings are a stale skill contract (SPEC-SKILL-CONTRACT section assignments) and a missing `verdict` enum value that causes a schema-agent semantic gap.

---

## Severity Definitions

| Severity | Meaning |
|----------|---------|
| **Critical** | Pipeline will malfunction or produce wrong output |
| **Significant** | Incorrect cross-references or missing constraints that could mislead agents |
| **Moderate** | Missing validation, undefined fields, or inconsistencies with limited runtime impact |
| **Minor** | Documentation staleness, cosmetic issues, or theoretical edge cases |

---

## New Findings

### S-01 [Significant] — SPEC-SKILL-CONTRACT Section 6 is stale

**Files**: `.github/skills/SPEC-SKILL-CONTRACT.md` Section 6  
**Cross-ref**: `.github/agents/spec-architect.agent.md` Step 7b, individual `spec-*/SKILL.md` files

The SPEC-SKILL-CONTRACT's dispatch order table has wrong section assignments:

| Skill | Contract claims | Actual (Step 7b + SKILL.md) |
|-------|----------------|---------------------------|
| `spec-requirements` | Sections 4, 5 | Sections 4, 10, 12, 13 |
| `spec-user-stories` | Section 6 | Sections 5, 6 |

The contract omits sections 10, 12, and 13 from `spec-requirements` and misattributes section 5 (which belongs to `spec-user-stories`). The agent file (Step 7b) and individual SKILL.md files are mutually consistent — only the contract is stale.

**Fix**: Update SPEC-SKILL-CONTRACT Section 6 to match spec-architect.agent.md Step 7b.

---

### S-02 [Significant] — `verdict` enum missing `approved_with_findings`

**Files**: `.github/schemas/enums.yaml`, `.github/schemas/reviewer-to-orchestrator.schema.yaml`, `.github/agents/review-coordinator.agent.md`, `.github/skills/REVIEW-SKILL-CONTRACT.md` Section 8

The Review Coordinator produces three verdicts: `Approved`, `Approved with Findings`, `Changes Required`. Both `enums.yaml` and `reviewer-to-orchestrator.schema.yaml` only define two values: `approved`, `changes_required`. The third verdict (`approved_with_findings`) is undeclared.

Additionally, casing is inconsistent: the enum uses `snake_case` (`approved`, `changes_required`), while the agent uses Title Case ("Approved", "Changes Required"). No normalization step exists.

**Impact**: Schema validation for the Orchestrator handoff would reject a legitimate "Approved with Findings" verdict. The Review Coordinator and REVIEW-SKILL-CONTRACT Section 8 agree on 3 verdicts; the schema layer only knows 2.

**Fix**: Add `approved_with_findings` to `enums.yaml` verdict enum and to `reviewer-to-orchestrator.schema.yaml`. Document the casing convention.

---

### M-01 [Moderate] — `review_cycles` field has no schema definition

**Files**: All schema files, `.github/schemas/enums.yaml`  
**Used by**: `review-coordinator.agent.md` (increments), `orchestrator.agent.md` (reads `>= 2` stall check)

The `review_cycles` WP frontmatter field is used by two agents but defined in zero schemas. It has no declared type, no initial value contract (Review Coordinator says "if absent, add it with value 1"), and no validation rule. Any schema-based validation of WP frontmatter would miss this field entirely.

**Fix**: Define `review_cycles` in the WP frontmatter schema (or a new `wp-frontmatter.schema.yaml`) as type `integer`, minimum 0, default absent.

---

### M-02 [Moderate] — retro-spec has no Step 0 schema validation

**Files**: `.github/agents/retro-spec.agent.md`  
**Cross-ref**: `spec-architect.agent.md` Step 0, `coder.agent.md` Step 0

Both `spec-architect` and `coder` validate their inbound handoff schemas at Step 0 (checking `required_artifacts`, `required_state`, `context_fields`, `validation_rules`). `retro-spec` only checks that `codebase_path` is non-empty and that the directory exists — no schema validation whatsoever.

**Note**: This was NM-05 in R2, classified as minor. Elevating to moderate because it's the only coordinator agent without schema validation, creating a structural gap.

**Fix**: Add schema validation to retro-spec Step 0, consistent with spec-architect and coder.

---

### M-03 [Moderate] — `retro-patterns.md` referenced but does not exist

**Files**: `.github/skills/RETRO-SKILL-CONTRACT.md` input #9, `.github/agents/retro-spec.agent.md` dispatch templates  
**Search result**: Zero files matching `retro-patterns.md` anywhere in the workspace

The RETRO-SKILL-CONTRACT defines `patterns` as input #9 sourced from `retro-patterns.md`. The retro-spec agent's dispatch templates include a `<patterns>` placeholder. But no step in the agent reads any `retro-patterns.md` file, and the file does not exist. The placeholder is always empty.

**Impact**: All retro skills receive an empty `patterns` input, bypassing the anti-pattern system entirely. If patterns were ever curated for retro extractions, they would be silently ignored.

**Fix**: Either create `retro-patterns.md` and add a read step in retro-spec.agent.md, or remove the patterns input from the RETRO-SKILL-CONTRACT and dispatch templates if retro patterns are not planned.

---

### M-04 [Moderate] — 7 handoff paths lack schemas

**Files**: `.github/agents/*.agent.md` frontmatter `handoffs:` entries, `.github/schemas/`

| Agent | Handoff | Missing schema |
|-------|---------|----------------|
| `1.1. Brainstorming` | Develop into Specification | `brainstorming-to-spec.schema.yaml` |
| `1.1. Brainstorming` | Continue as Standard Ideation | `brainstorming-to-ideation.schema.yaml` |
| `2. Spec Architect` | Return to Ideation | `spec-architect-to-ideation.schema.yaml` |
| `4. Coder` | Clarify Specification | `coder-to-spec.schema.yaml` |
| `4. Coder` | Add or Refine Tasks | `coder-to-planner.schema.yaml` |
| `6. Docs Agent` | Return to Coder | `docs-agent-to-coder.schema.yaml` |
| `6. Docs Agent` | Return to Review Coordinator | `docs-agent-to-reviewer.schema.yaml` |

These are declared handoffs in agent frontmatter with no corresponding schema file. Schema validation on these transitions is impossible.

**Impact**: Low in Orchestrator mode (the Orchestrator drives transitions, not handoffs). Higher if agents are used standalone where handoff buttons trigger directly.

**Fix**: Create schema files for each, or explicitly document that these handoffs are Orchestrator-bypassed and schema-exempt.

---

### M-05 [Moderate] — `spec-api-design` SKILL.md doesn't mention `interfaces.<ext>` artifact

**Files**: `.github/agents/spec-architect.agent.md` Step 7b, `.github/skills/spec-api-design/SKILL.md`

Step 7b lists three artifacts for `spec-api-design`: `api-contracts.<ext>`, `error-catalog.<ext>`, `interfaces.<ext>`. The skill's own SKILL.md only mentions the first two. The `interfaces.<ext>` artifact is undocumented in the skill that's supposed to produce it.

**Fix**: Either add `interfaces.<ext>` to spec-api-design/SKILL.md's artifact list, or remove it from Step 7b if it was merged into `api-contracts.<ext>`.

---

### N-01 [Minor] — Research file cleanup is not error-guarded

**Files**: `.github/agents/ideation.agent.md` (line ~114-116), `.github/agents/brainstorming.agent.md` (line ~75-77)

Both agents treat research file cleanup as a post-success step. If the research skill dispatch fails after partially writing `.sdd/research-{timestamp}.md`, the cleanup instruction is never reached. The failure branches say "log the failure and proceed" but don't include `Remove-Item`.

**Fix**: Add cleanup to both success and failure branches (try/finally semantics), or add a catch-all cleanup at the end of each round.

---

### N-02 [Minor] — Ideation wildcard cleanup could delete brainstorming research files

**Files**: `.github/agents/ideation.agent.md` (line ~114)

Ideation uses `Remove-Item .sdd/research-*.md` (wildcard). Brainstorming writes to the same `.sdd/research-{timestamp}.md` pattern. If both agents run concurrently in the same workspace, ideation's wildcard cleanup could delete brainstorming's in-progress research file.

**Fix**: Use specific file path cleanup (like brainstorming does) instead of wildcard.

---

### N-03 [Minor] — `ideation-to-spec.schema.yaml` uses `{NNN}-*.md` glob

**Files**: `.github/schemas/ideation-to-spec.schema.yaml` `required_artifacts`

The `required_artifacts` path pattern is `.sdd/ideas/{NNN}-*.md` where `{NNN}` is defined as `\d{2,3}`. This means ideation brief filenames must start with 2-3 digits. This constraint is not documented in ideation.agent.md's naming convention and may silently reject valid briefs with longer/shorter numeric prefixes.

**Impact**: Low — the pattern is informational, not enforced at runtime.

**Fix**: Align the pattern with ideation.agent.md's actual naming scheme, or document the constraint.

---

### N-04 [Minor] — `review-spec-completeness` not wired in Planner

**Files**: `.github/agents/planner.agent.md`, `.github/skills/review-spec-completeness/SKILL.md`

The REVIEW-SKILL-CONTRACT Section 5 states `review-spec-completeness` is "dispatched separately by the Planner as a pre-planning validation." The skill's own SKILL.md says "dispatched by the Review Coordinator or Planner coordinator during its completeness pre-check." But `planner.agent.md` contains zero references to `review-spec-completeness` — no import, no dispatch, no mention.

**Impact**: The pre-planning spec completeness gate described in the contract is not implemented in the Planner.

**Fix**: Add a completeness pre-check step to planner.agent.md that dispatches `review-spec-completeness` before Phase 1, or update the contract and SKILL.md to remove the Planner dispatch claim if this was deprioritized.

---

### N-05 [Minor] — Semantic-commit skill unused by all agents

**Files**: `.github/skills/semantic-commit/SKILL.md`, all `.github/agents/*.agent.md` files

The `semantic-commit` skill exists and is referenced by `commit-changes.prompt.md`, but zero agents dispatch it. Every agent implements its own ad-hoc `git add` + `git commit` steps with inline commit message formatting. This duplicates logic across 6+ agents and means the semantic-commit skill's grouping algorithm is never used during pipeline execution.

**Impact**: Cosmetic — agents commit successfully with inline logic. But commit quality may be inconsistent across agents.

**Fix**: Either integrate semantic-commit into agent commit workflows, or explicitly document that it's a user-facing utility only (not part of the pipeline).

---

### N-06 [Minor] — `docs_completed` only validated in handoff schema, not WP frontmatter schema

**Files**: `.github/schemas/docs-agent-to-orchestrator.schema.yaml`, no WP frontmatter schema exists

The `docs_completed` boolean is validated as a context field in the Docs Agent → Orchestrator handoff schema. But the Orchestrator also reads this field directly from WP frontmatter during state re-reads (e.g., after session restart). There is no WP frontmatter schema that governs this field on the WP file itself.

**Impact**: Low — the field is binary (true/false) and set by a single agent, making corruption unlikely.

**Fix**: Consider creating a WP frontmatter schema, or document that `docs_completed` is a pipeline-internal field validated only at handoff time.

---

## Appendix A — R2 Fix Verification

All 15 R2 issues have been verified as resolved:

| ID | Issue | Status |
|----|-------|--------|
| NC-01 | docs-agent missing doc-configuration, doc-deployment | **FIXED** — positions 5, 6 in Step 5 |
| NC-02 | review-coordinator order vs REVIEW-SKILL-CONTRACT | **FIXED** — both tables match exactly |
| NS-01 | Silent approval on critical skill dispatch failure | **FIXED** — review-spec/review-security failures → FAIL |
| NS-02 | Infinite coverage remediation loop | **FIXED** — capped at 2 re-dispatches |
| NS-03 | Missing `lane: blocked` in Decision Table | **FIXED** — row 9 added |
| NS-04 | Missing test status in Review Coordinator prompt | **FIXED** — "Test status: check WP Activity Log" |
| NS-05 | Stall detection threshold mismatch | **FIXED** — "2 consecutive rounds" |
| NM-01 | REVIEW-SKILL-CONTRACT missing "Approved with Findings" | **FIXED** — Section 8 now has 3 verdicts |
| NM-02 | Three-way ordering inconsistency | **FIXED** — batch table matches canonical order |
| NM-03 | `review_cycles` stall not wired in Orchestrator | **FIXED** — Step 5 Priority 2 item 2 |
| NM-04 | Research file accumulation | **FIXED** — both agents have cleanup (but see N-01) |
| NM-05 | retro-spec no schema validation | **STILL MISSING** — elevated to M-02 |
| NM-06 | Pattern file version tracking | Verified present in agents |
| NM-07 | Coder unquoted YAML frontmatter | **FIXED** — `name: "4. Coder"` |
| NM-08 | Brainstorming 10-round minimum override | **FIXED** — enforces minimum before round 10 |

---

## Summary

| Severity | Count | IDs |
|----------|-------|-----|
| Critical | 0 | — |
| Significant | 2 | S-01, S-02 |
| Moderate | 5 | M-01 through M-05 |
| Minor | 6 | N-01 through N-06 |
| **Total** | **13** | |

### Priority Fix Order

1. **S-01** — Update SPEC-SKILL-CONTRACT Section 6 (stale section assignments)
2. **S-02** — Add `approved_with_findings` to verdict enum and schema
3. **M-05** — Resolve `interfaces.<ext>` artifact mismatch
4. **M-03** — Decide on retro-patterns.md (create or remove references)
5. **M-02** — Add schema validation to retro-spec Step 0
6. **M-01** — Define `review_cycles` in schema layer
7. **M-04** — Create missing handoff schemas (or document exemptions)
8. Remaining minor items at discretion
