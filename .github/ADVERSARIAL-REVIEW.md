# Adversarial Review Report — `.github/` Artifacts

**Date**: 2025-01-27  
**Scope**: All agents, skills, schemas, contracts, and supporting files in `.github/`  
**Method**: Full cross-reference validation, structural consistency audit, and semantic correctness review

---

## Executive Summary

The SDD agent ecosystem is **architecturally sound** with a well-designed pipeline (Ideation → Spec → Plan → Code → Review → Docs) and strong contract-first principles. However, a comprehensive adversarial audit reveals **4 critical gaps**, **12 significant issues**, and **~20 minor inconsistencies** that collectively risk pipeline failures, silent data loss, and review blind spots.

**Verdict**: The system would fail under adversarial stress testing. The critical gaps must be fixed before reliable autonomous operation.

---

## Table of Contents

1. [What's Good](#1-whats-good)
2. [Critical Gaps](#2-critical-gaps)
3. [Significant Issues](#3-significant-issues)
4. [Minor Inconsistencies](#4-minor-inconsistencies)
5. [Per-Family Analysis](#5-per-family-analysis)
6. [Cross-Cutting Concerns](#6-cross-cutting-concerns)
7. [Recommendations](#7-recommendations)

---

## 1. What's Good

### Architecture & Design
- **Contract-first approach**: Skills consume typed contracts from `.sdd/plans/contracts/`, ensuring implementation matches specification. This is the system's strongest design decision.
- **Skill contracts**: 5 well-defined skill contracts (CODER, PLAN, SPEC, RETRO, DOC) enforce consistent I/O patterns across skill families.
- **Accumulator pattern**: Review skills accumulate findings in a structured format that enables machine-readable verdicts — excellent for automation.
- **Topological WP ordering**: The Orchestrator uses Kahn's algorithm with cycle detection for dependency-aware WP scheduling. Robust.
- **State verification protocol**: Cross-verifying `state.md` against WP frontmatter on startup prevents stale state drift.
- **Error escalation chain**: 3-tier failure handling (retry → escalate → halt) with error logging prevents infinite loops.

### Individual Highlights
- **`coder.agent.md`**: Best-in-class commit policy, comprehensive workflow, clear skill dispatch sequence.
- **`orchestrator.agent.md`**: Thorough state machine documentation, valid transition table, decision table with 12 conditions.
- **`review-architecture/SKILL.md`**: Solid SOLID principles checklist, practical "check → report" structure.
- **`doc-architecture/SKILL.md`**: Best doc skill — 8-item quality checklist, clear section template.
- **`research/SKILL.md`**: Excellent security posture — treats all web content as untrusted, includes prompt injection defense.
- **`semantic-commit/SKILL.md`**: Principled approach to atomic commits with clear grouping rules.

### Structural Consistency
- All agents follow the `.agent.md` pattern with YAML frontmatter, tools, instructions, and handoffs.
- All skills follow the `SKILL.md` pattern with inputs, workflow steps, and output format.
- Handoff schemas use a consistent YAML structure with `source_agent`, `target_agent`, and `context_fields`.

---

## 2. Critical Gaps

### CRIT-01: No REVIEW-SKILL-CONTRACT.md Exists

**Impact**: All 9 review skills operate without a governing contract, unlike every other skill family.  
**Risk**: Review skills diverge in structure, severity definitions, output format, and workflow steps — and they already have.  
**Evidence**: review-performance has 6 steps while peers have 7. review-spec-completeness outputs plain markdown while peers output YAML frontmatter. Severity tables vary across skills.  
**Fix**: Create `REVIEW-SKILL-CONTRACT.md` defining: mandatory workflow steps, output YAML frontmatter schema, severity enum, quality checklist requirements, and WP file read obligation.

### CRIT-02: review-docs Checks for Files No Skill Produces

**Impact**: review-docs validates 7 "standard documentation files" but 2 of them (`configuration-guide.md`, `deployment-guide.md`) are never produced by any doc skill.  
**Risk**: Every review-docs run will report missing files as failures — creating permanent false negatives that erode trust in the review pipeline.  
**Evidence**: The 6 doc skills produce: `api-reference.md`, `architecture.md`, `CHANGELOG.md`, `developer-guide.md`, `user-guide.md`, and inline code docs. No skill produces `configuration-guide.md` or `deployment-guide.md`.  
**Fix**: Either (a) remove these from review-docs' checklist, or (b) create doc-configuration and doc-deployment skills.

### CRIT-03: doc-inline-code Creates TODOs That review-quality Flags

**Impact**: doc-inline-code inserts `// TODO: ...` markers for unclear documentation. review-quality flags TODO markers as WARN findings.  
**Risk**: Every doc-inline-code run generates review findings on the next review cycle, creating a circular escalation loop.  
**Fix**: Use a different marker (e.g., `// DOCFIX:`) that review-quality doesn't flag, or add a review-quality exception for doc-generated markers.

### CRIT-04: `{WP-slug}` Placeholder Undefined in planner-to-coder Schema

**Impact**: `planner-to-coder.schema.yaml` uses `{WP-slug}` in `required_artifacts` paths but only defines `{NN}` and `{slug}` in `placeholder_patterns`.  
**Risk**: Any validation against this schema will fail on the `{WP-slug}` pattern. The entire Planner→Coder handoff path is broken at the schema level.  
**Fix**: Add `{WP-slug}` to `placeholder_patterns` or change the artifact path to use `{slug}`.

---

## 3. Significant Issues

### SIG-01: 3 Schemas Reference Wrong Orchestrator Name

| Schema | Field | Has | Should Be |
|--------|-------|-----|-----------|
| `reviewer-to-orchestrator.schema.yaml` | `target_agent` | `"1. Orchestrator"` | `"0. Orchestrator"` |
| `coder-complete-to-orchestrator.schema.yaml` | `target_agent` | `"1. Orchestrator"` | `"0. Orchestrator"` |
| `docs-agent-to-orchestrator.schema.yaml` | `target_agent` | `"1. Orchestrator"` | `"0. Orchestrator"` |

### SIG-02: reviewer-to-orchestrator Also Has Wrong Source Name

`source_agent: "5. Reviewer"` should be `"5. Review Coordinator"`. This schema has **two** name errors.

### SIG-03: 8 Agent Handoffs Lack Corresponding Schema Files

| Source → Target | Missing Schema |
|-----------------|----------------|
| Brainstorming → Spec Architect | `brainstorming-to-spec.schema.yaml` |
| Brainstorming → Ideation | `brainstorming-to-ideation.schema.yaml` |
| Coder → Spec Architect | `coder-to-spec.schema.yaml` |
| Coder → Planner | `coder-to-planner.schema.yaml` |
| Docs Agent → Coder | `docs-agent-to-coder.schema.yaml` |
| Docs Agent → Review Coordinator | `docs-agent-to-reviewer.schema.yaml` |
| Review Coordinator → Planner | `reviewer-to-planner.schema.yaml` |
| Spec Architect → Ideation | `spec-to-ideation.schema.yaml` |

### SIG-04: 3 `*-to-orchestrator` Schemas Are Orphans

No agent defines a handoff button routing to the Orchestrator. These schemas exist but are unreachable from the agent graph. They may be intended as internal pipeline signals, but that pattern is undocumented.

### SIG-05: 4/9 Review Skills Missing WP File Read Step

`review-deps`, `review-performance`, `review-quality`, and `review-security` don't read the WP file for context before reviewing. They jump straight to code analysis without knowing what the WP is supposed to implement.

### SIG-06: coder-to-reviewer Schema Missing `base_schema` Reference

`coder-to-reviewer.schema.yaml` doesn't declare `base_schema: ".github/schemas/base-handoff.schema.yaml"` despite being from the same source agent as `coder-complete-to-orchestrator.schema.yaml` which does.

### SIG-07: review-spec `warn: <count>` Template Contradicts Rule

The output template includes `warn: <count>` but the skill's own rules say `warn` is always 0 for spec adherence. The template should show `warn: 0`.

### SIG-08: doc-developer-guide Template Truncated

Section 3 output template has a "Formatting" subsection heading with no body content. The template is incomplete.

### SIG-09: doc-user-guide References Nonexistent Contract Files

References `config-schema.<ext>` contract files that no plan skill or spec skill defines or produces.

### SIG-10: review-spec-completeness Output Format Diverges

Uses plain markdown output with no YAML frontmatter, unlike every other review skill. This breaks any tooling that expects a consistent review output format.

### SIG-11: Severity Table Inconsistencies Across Review Skills

- `review-spec` uses INFO severity in Section 13 but doesn't define it in severity rules
- `review-spec-completeness` references LOW severity but never assigns it
- `review-deps` has no PASS severity defined
- No canonical severity enum exists (unlike `lane` which has `enums.yaml`)

### SIG-12: doc-inline-code Receives `docs_dir` Input But Never Uses It

The skill declares `docs_dir` as an input parameter but no workflow step references it.

---

## 4. Minor Inconsistencies

| # | File | Issue |
|---|------|-------|
| M-01 | `review-architecture` | Missing quality checklist (most review skills have one) |
| M-02 | `review-architecture` | References FR numbers without a contract file defining them |
| M-03 | `review-performance` | Only 6 workflow steps (peers have 7) |
| M-04 | `review-security` | OWASP numbering inconsistency between examples and checklist |
| M-05 | `review-tests` | Branch coverage threshold (90%) > code coverage (80%) — atypical |
| M-06 | `doc-api-reference` | No quality checklist (peer `doc-architecture` has one) |
| M-07 | `doc-api-reference` | Ambiguous `spec_path` description |
| M-08 | `doc-changelog` | Rule/template contradiction for trailing `---` separator |
| M-09 | `doc-changelog` | No "No Updates" handling when WP has no changelog-worthy changes |
| M-10 | `ideation-to-spec.schema.yaml` | Uses bare `*` glob instead of named placeholder |
| M-11 | `base-handoff.schema.yaml` | Utility schema with no direct agent binding (expected but undocumented) |
| M-12 | No agent references `semantic-commit` skill | Commit policies mention committing but don't dispatch the skill |
| M-13 | `review-deps` | Uses `#tool:web` which is environment-dependent |
| M-14 | `review-docs` | Claims "6 standard documentation files" but lists 7 |
| M-15 | Retro-spec skills | No issues found — well-structured, consistent with RETRO-SKILL-CONTRACT.md |

---

## 5. Per-Family Analysis

### 5.1 Agents (9 files)

| Agent | Quality | Key Issue |
|-------|---------|-----------|
| `0. Orchestrator` | ★★★★☆ | Excellent state machine; commit policy added this session |
| `1. Ideation` | ★★★★☆ | Clean workflow; good commit policy |
| `1.1. Brainstorming` | ★★★☆☆ | Functional but 2 missing handoff schemas |
| `2. Spec Architect` | ★★★★☆ | Solid; commit policy upgraded this session |
| `3. Planner` | ★★★★☆ | Solid; commit policy upgraded this session |
| `4. Coder` | ★★★★★ | Best agent — comprehensive workflow, strong contract integration |
| `5. Review Coordinator` | ★★★★☆ | Good skill dispatch; commit policy added this session |
| `6. Docs Agent` | ★★★☆☆ | Functional but commit policy was missing until this session |
| `7. Retro-Spec` | ★★★★☆ | Well-designed extraction pipeline |

### 5.2 Code Skills (5 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `code-debug` | ★★★★☆ | Solid diagnostic workflow |
| `code-env-setup` | ★★★★☆ | Good baseline verification |
| `code-implementation` | ★★★★★ | Best code skill — contract-first, business logic extraction added |
| `code-integration-tests` | ★★★★☆ | Good boundary testing guidance |
| `code-unit-tests` | ★★★★☆ | Clear coverage thresholds |

### 5.3 Plan Skills (5 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `plan-acceptance` | ★★★★☆ | Business logic criteria added this session |
| `plan-api-contracts` | ★★★★☆ | Clean contract generation |
| `plan-cross-wp-validation` | ★★★★☆ | Good cross-WP consistency checks |
| `plan-decomposition` | ★★★★☆ | Business logic awareness added this session |
| `plan-state-machines` | ★★★★☆ | Solid state enum generation |

### 5.4 Spec Skills (7 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `spec-requirements` | ★★★★★ | Deep business logic section added this session |
| `spec-user-stories` | ★★★★☆ | Business logic scenario rule added |
| `spec-api-design` | ★★★★☆ | Clean contract output |
| `spec-architecture` | ★★★★☆ | Good tech stack guidance |
| `spec-data-model` | ★★★★☆ | Solid entity definitions |
| `spec-security` | ★★★★☆ | OWASP-aligned |
| `spec-test-strategy` | ★★★★☆ | BDD scenario mapping |
| `spec-traceability` | ★★★★☆ | Good cross-reference matrix |

### 5.5 Review Skills (9 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `review-architecture` | ★★★☆☆ | Missing quality checklist, no contract file |
| `review-deps` | ★★★☆☆ | Missing WP read step, no PASS severity, env-dependent |
| `review-docs` | ★★☆☆☆ | **CRIT-02** — checks for nonexistent files |
| `review-performance` | ★★★☆☆ | Missing WP read step, 6 steps not 7 |
| `review-quality` | ★★★☆☆ | Missing WP read step |
| `review-security` | ★★★☆☆ | Missing WP read step, OWASP numbering off |
| `review-spec` | ★★★☆☆ | warn template contradiction, INFO undefined |
| `review-spec-completeness` | ★★☆☆☆ | Output format diverges from all peers |
| `review-tests` | ★★★☆☆ | Unusual coverage thresholds |

**Family verdict**: Weakest family. No governing contract. Structural drift already visible.

### 5.6 Doc Skills (6 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `doc-api-reference` | ★★★☆☆ | No quality checklist |
| `doc-architecture` | ★★★★★ | Best doc skill — 8-item checklist |
| `doc-changelog` | ★★★☆☆ | Template contradictions, no empty-change handling |
| `doc-developer-guide` | ★★☆☆☆ | Truncated template |
| `doc-inline-code` | ★★☆☆☆ | **CRIT-03** — creates TODOs that trigger review warnings |
| `doc-user-guide` | ★★★☆☆ | References nonexistent contract files |

### 5.7 Retro Skills (7 files)

| Skill | Quality | Key Issue |
|-------|---------|-----------|
| `retro-discovery` | ★★★★☆ | Good codebase scanning |
| `retro-architecture` | ★★★★☆ | Solid extraction workflow |
| `retro-business-logic` | ★★★★★ | Deep functional understanding model |
| `retro-data-model` | ★★★★☆ | Good entity extraction |
| `retro-api-contracts` | ★★★★☆ | Clean interface extraction |
| `retro-cross-cutting` | ★★★★☆ | Good NFR extraction |
| `retro-test-analysis` | ★★★★☆ | Solid test-to-requirement mapping |
| `retro-assembly` | ★★★★☆ | Good multi-level spec assembly |

**Family verdict**: Strongest skill family. Consistent with RETRO-SKILL-CONTRACT.md. Recently built, benefits from lessons learned.

### 5.8 Schemas (14 files)

| Schema | Quality | Key Issue |
|--------|---------|-----------|
| `base-handoff.schema.yaml` | ★★★★☆ | Good base pattern, undocumented utility status |
| `enums.yaml` | ★★★★★ | Clean canonical enum source |
| `coder-to-reviewer` | ★★★☆☆ | Missing `base_schema` reference |
| `coder-complete-to-orchestrator` | ★★★☆☆ | Wrong Orchestrator name |
| `docs-agent-to-orchestrator` | ★★★☆☆ | Wrong Orchestrator name |
| `reviewer-to-orchestrator` | ★☆☆☆☆ | **Two** wrong names |
| `planner-to-coder` | ★★☆☆☆ | **CRIT-04** — undefined placeholder |
| `ideation-to-spec` | ★★★☆☆ | Undocumented glob pattern |
| Others (6) | ★★★★☆ | Clean |

### 5.9 Contracts (5 files)

| Contract | Quality | Key Issue |
|----------|---------|-----------|
| CODER-SKILL-CONTRACT.md | ★★★★★ | Comprehensive, well-enforced |
| PLAN-SKILL-CONTRACT.md | ★★★★☆ | Clean |
| SPEC-SKILL-CONTRACT.md | ★★★★☆ | Clean |
| RETRO-SKILL-CONTRACT.md | ★★★★☆ | Clean |
| DOC-SKILL-CONTRACT.md | ★★★★☆ | Clean |
| **REVIEW-SKILL-CONTRACT.md** | **MISSING** | **CRIT-01** |

---

## 6. Cross-Cutting Concerns

### 6.1 Semantic-Commit Skill Not Referenced

No agent dispatches the `semantic-commit` skill. All agents now have `<commit_policy>` blocks (fixed this session), but they describe commit *rules* without leveraging the skill that *implements* semantic grouping. The skill exists in isolation.

### 6.2 Review Pipeline Circular Risk

The doc-inline-code → TODO → review-quality → WARN → fix cycle (CRIT-03) is a systemic risk that could stall WPs in perpetual review loops, eventually triggering the Orchestrator's review cycle stall detector (review_cycles >= 3).

### 6.3 Schema Coverage Gap

Only 6 of 14 possible agent-to-agent handoffs have schemas defined. The 8 missing schemas mean those handoff paths have no formal contract validation.

### 6.4 FR References Without Contract Backing

Multiple review skills reference "FR-039", "FR-040" etc. without these being defined in any contract file. The FRs exist in the spec, but skills should reference a contract file that contains their extracted subset.

---

## 7. Recommendations

### Priority 1 — Fix Before Next Pipeline Run

| # | Action | Effort |
|---|--------|--------|
| 1 | Create `REVIEW-SKILL-CONTRACT.md` with mandatory structure (CRIT-01) | Medium |
| 2 | Fix `review-docs` file list to match actual doc skill outputs (CRIT-02) | Small |
| 3 | Change `doc-inline-code` TODO markers to `DOCFIX:` (CRIT-03) | Small |
| 4 | Fix `{WP-slug}` placeholder in `planner-to-coder.schema.yaml` (CRIT-04) | Small |
| 5 | Fix 3 wrong Orchestrator names in schemas (SIG-01) | Small |
| 6 | Fix wrong source name in `reviewer-to-orchestrator` (SIG-02) | Small |

### Priority 2 — Fix Soon

| # | Action | Effort |
|---|--------|--------|
| 7 | Add WP file read step to 4 review skills (SIG-05) | Medium |
| 8 | Fix `review-spec` warn template (SIG-07) | Small |
| 9 | Complete `doc-developer-guide` template (SIG-08) | Small |
| 10 | Normalize review skill output format (SIG-10, SIG-11) | Medium |
| 11 | Add `base_schema` to `coder-to-reviewer` (SIG-06) | Small |
| 12 | Remove unused `docs_dir` from `doc-inline-code` (SIG-12) | Small |
| 13 | Remove `config-schema` reference from `doc-user-guide` (SIG-09) | Small |

### Priority 3 — Improve When Convenient

| # | Action | Effort |
|---|--------|--------|
| 14 | Create 8 missing handoff schemas (SIG-03) | Large |
| 15 | Add `severity` enum to `enums.yaml` for review skills | Small |
| 16 | Add quality checklists to review skills that lack them | Medium |
| 17 | Document the `*-to-orchestrator` schema pattern (SIG-04) | Small |
| 18 | Wire `semantic-commit` skill into agent commit policies | Medium |

---

## Scorecard

| Family | Files | Avg Quality | Critical Issues | Significant Issues |
|--------|-------|-------------|-----------------|-------------------|
| Agents | 9 | ★★★★☆ | 0 | 0 |
| Code Skills | 5 | ★★★★☆ | 0 | 0 |
| Plan Skills | 5 | ★★★★☆ | 0 | 0 |
| Spec Skills | 8 | ★★★★☆ | 0 | 0 |
| Review Skills | 9 | ★★★☆☆ | 2 | 4 |
| Doc Skills | 6 | ★★★☆☆ | 1 | 3 |
| Retro Skills | 8 | ★★★★☆ | 0 | 0 |
| Schemas | 14 | ★★★☆☆ | 1 | 4 |
| Contracts | 5+1 | ★★★★☆ | 1 (missing) | 0 |
| **TOTAL** | **69** | **★★★½☆** | **4** | **12** |

---

*Report generated by adversarial review of all 69 artifacts in `.github/`. Findings are actionable and prioritized by pipeline impact.*
