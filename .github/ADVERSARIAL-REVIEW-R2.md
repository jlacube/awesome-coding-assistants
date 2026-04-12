# Adversarial Review Report — Round 2

**Date**: 2025-04-11  
**Scope**: All agents, skills, schemas, contracts, and supporting files in `.github/`  
**Baseline**: All 4 critical and 12 significant issues from the [Round 1 review](ADVERSARIAL-REVIEW.md) (2025-01-27) have been verified as **FIXED**  
**Method**: Full cross-reference validation, structural consistency audit, semantic correctness review, edge-case analysis, and infinite-loop/state-corruption analysis

---

## Executive Summary

Round 1 found 4 critical gaps, 12 significant issues, and ~20 minor inconsistencies — **all are now resolved**. This Round 2 review uncovers **2 new critical gaps**, **5 new significant issues**, and **8 new minor inconsistencies**. The findings concentrate in two areas: (1) agent-vs-contract ordering mismatches introduced when contracts were created without updating agents, and (2) missing guard rails for edge cases (infinite loops, undefined lane values, silent review failures).

**Verdict**: The system is substantially improved from Round 1 but would still exhibit incorrect behavior under specific failure scenarios. The two critical gaps produce wrong skill execution order in every run (not just edge cases).

---

## Table of Contents

1. [Previous Issues — Verification](#1-previous-issues--verification)
2. [Critical Gaps](#2-critical-gaps)
3. [Significant Issues](#3-significant-issues)
4. [Minor Inconsistencies](#4-minor-inconsistencies)
5. [Summary Table](#5-summary-table)
6. [Recommendations](#6-recommendations)

---

## 1. Previous Issues — Verification

All items from Round 1 have been re-checked against the current files:

| ID | Description | Status |
|----|-------------|--------|
| CRIT-01 | No REVIEW-SKILL-CONTRACT.md exists | **FIXED** — full contract now at `.github/skills/REVIEW-SKILL-CONTRACT.md` |
| CRIT-02 | review-docs checks for files no skill produces | **FIXED** — `doc-configuration` and `doc-deployment` skills now exist |
| CRIT-03 | doc-inline-code creates TODOs that review-quality flags | **FIXED** — now uses `# DOCFIX:` marker |
| CRIT-04 | `{WP-slug}` placeholder undefined in schema | **FIXED** — placeholder pattern added to `planner-to-coder.schema.yaml` |
| SIG-01 | 3 schemas reference wrong Orchestrator name | **FIXED** — all use `"0. Orchestrator"` |
| SIG-02 | reviewer-to-orchestrator has wrong source name | **FIXED** — now `"5. Review Coordinator"` |
| SIG-05 | 4/9 review skills missing WP file read step | **FIXED** — all 4 now have WP read step |
| SIG-06 | coder-to-reviewer schema missing base_schema | **FIXED** — field added |
| SIG-10 | review-spec-completeness output format diverges | **FIXED** — now produces YAML frontmatter |
| SIG-12 | doc-inline-code receives unused docs_dir input | **FIXED** — parameter removed |

---

## 2. Critical Gaps

### NC-01: docs-agent.agent.md Step 5 Missing 2 Skills from Canonical Order

**Files**: `docs-agent.agent.md` (Step 5, lines 115-120), `DOC-SKILL-CONTRACT.md` (Section 5, lines 78-88)

The agent's canonical dispatch table lists **6 skills**:

| Agent Position | Skill |
|----------------|-------|
| 1 | `doc-architecture` |
| 2 | `doc-api-reference` |
| 3 | `doc-user-guide` |
| 4 | `doc-developer-guide` |
| 5 | `doc-changelog` |
| 6 | `doc-inline-code` |

The governing contract (DOC-SKILL-CONTRACT.md Section 5) lists **8 skills**:

| Contract Position | Skill |
|-------------------|-------|
| 1 | `doc-architecture` |
| 2 | `doc-api-reference` |
| 3 | `doc-user-guide` |
| 4 | `doc-developer-guide` |
| **5** | **`doc-configuration`** |
| **6** | **`doc-deployment`** |
| 7 | `doc-changelog` |
| 8 | `doc-inline-code` |

Both `doc-configuration/SKILL.md` and `doc-deployment/SKILL.md` exist on disk (they were created to fix CRIT-02). Because they are absent from the agent's canonical list, Step 5b's fallback rule applies: "Skills discovered but NOT in the canonical list: dispatch AFTER all canonical skills, in alphabetical order." They execute after `doc-inline-code` instead of between `doc-developer-guide` and `doc-changelog`.

**Impact**: Configuration and deployment docs are generated *after* source code annotation and the changelog — violating the contract's intended dependency order where configuration/deployment docs inform the changelog.

**Fix**: Add `doc-configuration` (position 5) and `doc-deployment` (position 6) to the Step 5 table, shifting `doc-changelog` to 7 and `doc-inline-code` to 8.

---

### NC-02: Review Coordinator Dispatch Order Contradicts REVIEW-SKILL-CONTRACT

**Files**: `review-coordinator.agent.md` (Step 6, lines 189-196), `REVIEW-SKILL-CONTRACT.md` (Section 5, lines 130-137)

| Position | Agent Step 6 | Contract Section 5 |
|----------|-------------|-------------------|
| 1 | `review-spec` | `review-spec` ✓ |
| 2 | `review-security` | `review-architecture` ✗ |
| 3 | `review-quality` | `review-security` ✗ |
| 4 | `review-tests` | `review-quality` ✗ |
| 5 | `review-architecture` | `review-performance` ✗ |
| 6 | `review-performance` | `review-tests` ✗ |
| 7 | `review-docs` | `review-deps` ✗ |
| 8 | `review-deps` | `review-docs` ✗ |

Only position 1 matches. The REVIEW-SKILL-CONTRACT was created after the agent (to fix CRIT-01) as the governing contract (FR-029), but the agent was never updated to match.

**Impact**: The batch dispatch structure in Step 7d uses the agent's order for grouping, placing `review-architecture` in position 5 despite the contract designating it as position 2 (high priority). The contract's ordering rationale (architecture early to catch structural issues before detailed reviews) is defeated.

**Fix**: Update `review-coordinator.agent.md` Step 6 canonical order table to match `REVIEW-SKILL-CONTRACT.md` Section 5 exactly.

---

## 3. Significant Issues

### NS-01: Skill Dispatch Failure Silently Approves WP — Including Security Reviews

**File**: `review-coordinator.agent.md` (Steps 7e, 8.3, 10)

When a skill dispatch fails (Step 7e), the coordinator records a `WARN` finding with ID `DISPATCH-<skill-name>`. When a skill produces no findings file (Step 8.3), another `WARN` is recorded. In Step 10, zero `FAIL` + one or more `WARN` = verdict `Approved with Findings` → Step 13a sets `lane: done`.

A crashed `review-security` or `review-spec` subagent produces only WARNs, which results in WP approval. There is no distinction between "skill passed all checks" and "skill never ran."

**Fix**: Dispatch failures of `review-spec` and `review-security` (at minimum) should produce `FAIL` findings, since unevaluated correctness/security is not equivalent to passing.

---

### NS-02: Coverage Remediation Has No Max-Attempt Limit — Infinite Loop Risk

**File**: `coder.agent.md` (Step 9, line 475)

Step 9 says: "If coverage is below thresholds, re-dispatch the test skills (`code-unit-tests`, `code-integration-tests`) to add more tests, then re-check." No loop counter. No max attempts. No escape hatch.

Compare to Step 7 (debug loop): explicitly capped at 3 attempts with escalation on the 4th.

If the code has genuinely uncoverable paths (OS conditionals, generated code, language-level dead branches), coverage will never reach threshold and this step loops indefinitely.

**Fix**: Add `coverage_attempt = 1`, cap at 2 re-dispatches, escalate to user with full coverage report on the 3rd failure.

---

### NS-03: Orchestrator Has No Decision Table Row for `lane: blocked`

**Files**: `orchestrator.agent.md` (Decision Table rows 1-12), `enums.yaml` (line 15), `review-coordinator.agent.md` (stalled_cycle_escalation)

The Review Coordinator's stall escalation sets `lane: blocked`. The `blocked` value is in the `lane` enum. But the Orchestrator's 12-row Decision Table covers `planned`, `doing`, `for_review`, `to_do`, `done` — **not `blocked`**.

When the Orchestrator reads a WP with `lane: blocked`, it matches no routing row. Priority 2 routing falls through all WP-specific conditions, Priority 3 completion checks don't match either. The Orchestrator enters an undefined state — likely looping endlessly or falling to Row 1 ("No ideas, no specs, no plans").

**Fix**: Add Decision Table row: "WP has `lane: blocked` → escalate to user with blocking reason from WP frontmatter."

---

### NS-04: Orchestrator Prompt Template Omits Test Status Required by Reviewer's Step 0

**Files**: `orchestrator.agent.md` (Step 6), `review-coordinator.agent.md` (Step 0.3)

Review Coordinator Step 0.3 requires: "Verify all tests are passing (check the **handoff prompt** for test status)."

The Orchestrator's prompt template for the Review Coordinator: `"Review {wp_id}. It is at lane=for_review. WP file: {wp_path}. Spec: {spec_path}. Contracts: {contracts_dir}."` — no test status.

Step 0 validation must find test status in the handoff prompt, but the Orchestrator never includes it.

**Fix**: Either (a) add test status to the Orchestrator's Review Coordinator prompt template (e.g., "Tests: check WP Activity Log for test results"), or (b) change Step 0.3 to derive test status from the WP Activity Log instead of the handoff prompt.

---

### NS-05: "3 Consecutive Rounds" Stall Detector Uses 1-Round Lookback

**File**: `review-coordinator.agent.md` (`<stalled_cycle_escalation>` section)

The stall condition says "If any FB-XX items have persisted across **3 consecutive rounds**." The implementation says: "Read the **existing** `## Review` section (before overwriting)" and compare current FB-XX items against those. Step 12 overwrites the Review section, destroying all history except the immediately prior round.

Only the **previous** round's items are compared against — round N-2 and earlier are gone. The stall fires after just TWO consecutive identical failures (not three), contradicting the stated threshold. Conversely, if issues rotate (round 3 has FB-01, round 4 has FB-02, round 5 has FB-01), the stall never fires despite the WP oscillating.

**Fix**: Either (a) change the condition to "2 consecutive rounds" to match the actual 1-round lookback, or (b) read the last 3 Activity Log `review-coordinator` entries to compare across all three.

---

## 4. Minor Inconsistencies

### NM-01: REVIEW-SKILL-CONTRACT Section 8 Missing "Approved with Findings" Verdict

**File**: `REVIEW-SKILL-CONTRACT.md` — Section 8

Section 8 defines only `PASS` (zero FAILs → `done`) and `FAIL` (one or more FAILs → `to_do`). The intermediate `Approved with Findings` verdict (zero FAILs + WARNs → `done`) used by the Review Coordinator is undocumented in the governing contract.

---

### NM-02: Three-Way Ordering Conflict Across Canonical, Contract, and Batch Dispatch

**File**: `review-coordinator.agent.md` — Steps 6, 7d

Three different ordering systems exist:
- **Step 6 canonical**: spec, security, quality, tests, architecture, performance, docs, deps
- **Contract Section 5**: spec, architecture, security, quality, performance, tests, deps, docs
- **Step 7d batches**: {spec, tests}, {security, deps, architecture, performance}, {quality, docs}

Within each batch, internal execution order is unstated. No reconciliation note declares which ordering is authoritative.

---

### NM-03: Orchestrator Review-Cycle Stall Not Wired into Standard Routing

**Files**: `orchestrator-reference.md` (Step 8e), `orchestrator.agent.md` (Step 5, Decision Table)

Step 8e: "When `review_cycles >= 3`: Halt." But this check is in the error-recovery section (Steps 8c-8f) which fires only when `last_result` is `failed`. A Review Coordinator returning "Changes Required" sets `last_result: success` (it completed successfully). The stall check never fires through the standard routing path.

**Fix**: Add a guard in Step 5 Priority 2: "If `lane: for_review` AND `review_cycles >= 3` → escalate (Step 8e)."

---

### NM-04: Research File Accumulation Violates Agents' "MINIMIZE File Creation" Rule

**Files**: `ideation.agent.md`, `brainstorming.agent.md`

Both dispatch the Research Skill with: "write findings to `.sdd/research-{timestamp}.md`." Both have a rules entry: "MINIMIZE file creation." Research output files accumulate in `.sdd/` with timestamp names, are not in any commit_policy table, and are never cleaned up or referenced after the dispatch returns.

---

### NM-05: Retro-Spec Agent Has No Inbound Schema Validation Step

**File**: `retro-spec.agent.md`

All other main-pipeline agents (Coder, Review Coordinator, Planner, Spec Architect) validate incoming handoff schemas as Step 0. Retro-Spec's Step 0 is "Initialization and User Configuration" with no schema validation. Two outbound schema files exist but no inbound validation.

---

### NM-06: Review Round Counter Computed Twice Independently

**File**: `review-coordinator.agent.md` — Steps 7a and 11

Both steps independently count `review-coordinator` entries in the Activity Log. Step 7a uses the result to decide between "full review" and "re-review scoping" — if the values ever diverge (e.g., file re-read between steps), the wrong skill set is dispatched.

---

### NM-07: Coder Agent Unquoted YAML Frontmatter Value

**File**: `coder.agent.md` — frontmatter line 8

Uses unquoted `agent: 5. Review Coordinator` while every other agent file quotes the value (e.g., `agent: "5. Review Coordinator"`). YAML parses correctly but inconsistent style.

---

### NM-08: Brainstorming 10-Round Minimum Is Soft, Not Hard

**File**: `brainstorming.agent.md`

Rule: "You MUST sustain at least 10 rounds of Q&A." Enforcement: "if the user asks to wrap up early, confirm they are satisfied." A single "yes" confirmation bypasses the minimum after any number of rounds. Compare to Ideation's hard 5-round cap.

---

## 5. Summary Table

| ID | Severity | File | Description |
|----|----------|------|-------------|
| NC-01 | **Critical** | `docs-agent.agent.md` Step 5 | 2 skills missing from canonical order (`doc-configuration`, `doc-deployment`) |
| NC-02 | **Critical** | `review-coordinator.agent.md` Step 6 | 7/8 positions diverge from REVIEW-SKILL-CONTRACT Section 5 |
| NS-01 | **Significant** | `review-coordinator.agent.md` Steps 7e/8.3/10 | Crashed security/spec skill → WARN → WP approved |
| NS-02 | **Significant** | `coder.agent.md` Step 9 | Coverage remediation loop has no max-attempt cap |
| NS-03 | **Significant** | `orchestrator.agent.md` Decision Table | No routing row for `lane: blocked` |
| NS-04 | **Significant** | `orchestrator.agent.md` Step 6 | Prompt template omits test status required by Reviewer Step 0 |
| NS-05 | **Significant** | `review-coordinator.agent.md` stall detection | "3 consecutive rounds" uses 1-round lookback |
| NM-01 | Minor | `REVIEW-SKILL-CONTRACT.md` Section 8 | Missing "Approved with Findings" verdict |
| NM-02 | Minor | `review-coordinator.agent.md` Steps 6/7d | Three-way ordering conflict: canonical vs contract vs batch |
| NM-03 | Minor | `orchestrator.agent.md` Step 5 | `review_cycles >= 3` stall not wired into standard routing |
| NM-04 | Minor | `ideation.agent.md` + `brainstorming.agent.md` | Research files violate MINIMIZE file creation rule |
| NM-05 | Minor | `retro-spec.agent.md` | Only pipeline agent without Step 0 schema validation |
| NM-06 | Minor | `review-coordinator.agent.md` Steps 7a/11 | Round counter computed twice independently |
| NM-07 | Minor | `coder.agent.md` frontmatter | Unquoted YAML agent value |
| NM-08 | Minor | `brainstorming.agent.md` | 10-round minimum bypassed by single confirmation |

---

## 6. Recommendations

### Priority 1 — Fix Immediately (blocks correct execution on every run)
1. **NC-01**: Add `doc-configuration` and `doc-deployment` to `docs-agent.agent.md` Step 5 canonical order
2. **NC-02**: Align `review-coordinator.agent.md` Step 6 order with `REVIEW-SKILL-CONTRACT.md` Section 5

### Priority 2 — Fix Before Production Use (failure-mode bugs)
3. **NS-01**: Promote `review-spec` and `review-security` dispatch failures from WARN to FAIL
4. **NS-02**: Cap coverage remediation at 2 re-dispatches with escalation
5. **NS-03**: Add `lane: blocked` row to Orchestrator Decision Table
6. **NS-04**: Add test status to Orchestrator → Review Coordinator prompt, or change Step 0.3 to read Activity Log
7. **NS-05**: Fix stall detection to either match its "3 consecutive" claim or update the claim to "2 consecutive"

### Priority 3 — Clean Up (consistency and hygiene)
8. **NM-01**: Document the 3-way verdict in REVIEW-SKILL-CONTRACT Section 8
9. **NM-02**: Reconcile Step 6, Step 7d batches, and contract order into one authoritative sequence
10. **NM-03**: Wire `review_cycles >= 3` guard into Orchestrator Step 5 Priority 2
11. **NM-04–NM-08**: Series of consistency fixes
