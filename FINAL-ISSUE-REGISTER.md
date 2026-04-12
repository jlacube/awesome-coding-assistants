# Final Consolidated Issue Register

**Date**: 2026-04-12
**Scope**: `.github/agents/`, `.github/skills/`, `.github/schemas/`, `.github/prompts/`
**Purpose**: Merge all findings from adversarial reviews R1–R4 into a single actionable register, grouped by fix type for bulk resolution. Replaces the round-by-round review cycle.

---

## Definition of Done

This register is **closed** when:
- All `FIX` items are resolved
- All `ACCEPT` items have been reviewed and acknowledged
- `DEFER` items are logged but do not block closure

No Round 5 is needed after this.

---

## Honest Severity Reclassification

R4 inflated severities. This register applies corrected classifications:

| Level | Meaning |
|-------|---------|
| **FIX** | Must change — causes runtime failure, semantic contradiction, or silently wrong behavior on a standard pipeline path |
| **ACCEPT** | Real inconsistency, but has no practical runtime impact. Document or acknowledge, don't change |
| **DEFER** | Valid improvement, but not worth blocking closure. Track for future cleanup |

---

## Group A: Agent Contradictions and Logic Errors

*These are internal contradictions within agent files that could cause wrong behavior in standard flows. Each is a small targeted edit to a single `.agent.md` file.*

| # | ID | File(s) | Issue | Disposition | Fix |
|---|---|---|---|---|---|
| A1 | R4-C05 | `orchestrator.agent.md`, `orchestrator-reference.md` | `review_cycles` threshold: agent says `>= 2` (3 places), reference says `>= 3`, Failure Summary Table says both | **FIX** | Pick one (recommend `>= 2`). Update `orchestrator-reference.md` and the "3 times" row in the Failure Summary Table. |
| A2 | R4-S09 | `review-coordinator.agent.md` | Commit retry contradiction: Step 14g says "retry once, then continue to Step 15"; Step 15 says "halt, do not retry" | **FIX** | Unify: either both retry-then-continue or both halt. Recommend: retry once in both steps, halt only on second failure. |
| A3 | R4-S08 | `review-coordinator.agent.md` | Spec status promoted from `Draft → Approved`, bypassing the `Validated` checkpoint | **FIX** | Change Step 13a condition: only promote from `Validated → Approved`. Warn if `Draft` is found. |
| A4 | R4-S01 | `coder.agent.md` | Step 0 has no branch for `lane: doing` (resume after interruption) — schema validation halts | **FIX** | Add `lane: doing → skip schema validation` branch to Step 0. |
| A5 | R4-S12 | `coder.agent.md` | `lane: to_do` with no FB-XX items → enters Standard Mode, full re-run risks overwrite | **FIX** | Add guard: halt with error if `lane: to_do` and no FB-XX findings. |
| A6 | R4-M09 | `brainstorming.agent.md` | Rules refuse early wrap-up; Readiness Criteria allow waiver — contradiction | **FIX** | Pick one. Recommend: allow waiver in both sections. |
| A7 | R4-M06 | `orchestrator.agent.md` | Decision Table row 11 `State After: idle` is wrong for both user responses (should be `implementation` or `complete`) | **FIX** | Change `State After` to `varies` with a note: "yes → implementation, no → complete." |
| A8 | R4-M04 | `orchestrator-reference.md` | Corrupted state recovery omits `lane: to_do`, `lane: blocked`, mixed-lane priority | **FIX** | Add: `to_do → implementation`, `blocked → escalate`, and "highest-urgency lane wins" rule. |
| A9 | R4-S03 | `planner.agent.md` | Step 2 writes to `.sdd/plans/` before Step 6 creates the directory | **FIX** | Add `mkdir -p .sdd/plans/` at beginning of Step 2, or write completeness report to `.sdd/` root. |
| A10 | new-P02 | `planner.agent.md` | No error handling for `review-spec-completeness` skill failure in Step 2 (file absent, dispatch fails) | **FIX** | Add: "If the skill fails or the output file is absent, log a warning and proceed to Step 3." |
| A11 | new-P12 | `planner.agent.md` | Ambiguous-language violations routed to `plan-acceptance` but should go to `plan-decomposition` (wrong remediation skill) | **FIX** | Change Step 10c item 1: route task description issues to `plan-decomposition`. |
| A12 | R4-M11 | `review-coordinator.agent.md` | New pattern files created without `patterns_version` frontmatter → reload loop on every skill dispatch | **FIX** | Add `patterns_version: 1` to the initial pattern file template in Step 14b. |

**Effort**: 12 small targeted edits, one per agent file instance. No structural changes.

---

## Group B: Handoff and Schema Wiring

*These are about making the handoff/schema layer consistent. Currently many schemas exist but aren't actually validated. There are two strategies: (1) wire them in, or (2) remove them and accept that schemas are documentation-only for non-standard paths. Recommend option 2 for most.*

| # | ID | File(s) | Issue | Disposition | Fix |
|---|---|---|---|---|---|
| B1 | R4-C02 | `ideation.agent.md` | `send: true` handoff to Spec Architect omits `brief_path` (required by schema) | **FIX** | Add `brief_path` to handoff prompt. Alternatively, the Spec Architect should scan `.sdd/ideas/` as fallback when `brief_path` is absent — but the handoff fix is simpler. |
| B2 | R4-C03 | `brainstorming.agent.md` | Both `send: true` handoffs omit `brief_path` | **FIX** | Add `brief_path` to both handoff prompts. |
| B3 | new-P13 | `planner.agent.md` | `send: true` handoff to Coder missing `wp_path` and `spec_path` (required by `planner-to-coder` schema) | **FIX** | Add `wp_path` and `spec_path` to handoff prompt. |
| B4 | R4-S04 | `planner.agent.md` | Step 0 hardcodes `spec-to-planner.schema.yaml`, ignores `retro-spec-to-planner.schema.yaml` | **FIX** | Add conditional: if source is Retro-Spec, use `retro-spec-to-planner.schema.yaml`. |
| B5 | R4-M01 | `reviewer-to-coder.schema.yaml` | Missing `spec_path` and `contracts_dir` context fields (Coder needs both in rework mode) | **FIX** | Add `spec_path` (required) and `contracts_dir` (required) to the schema. |
| B6 | R4-M10 | `spec-architect.agent.md` | Step 0 lumps Brainstorming into Ideation path, bypassing `brainstorming-to-spec.schema.yaml` | **DEFER** | Both schemas have very similar content. The mismatch is pedantic. Add a comment: "Brainstorming uses the same schema as Ideation." |
| B7 | R4-S07 | 8 schema files + agents | 8 orphaned schemas: target agents never read them | **ACCEPT** | These schemas serve as documentation for non-standard paths (Coder→Planner, Docs→Coder, etc.) that the Orchestrator mediates. They don't need runtime validation — the Orchestrator controls routing. Add a header comment to each: `# Documentation-only: validated by Orchestrator routing, not by target agent Step 0`. |
| B8 | R4-S10 | `docs-agent.agent.md` | No Step 0 schema validation (only coordinator without one) | **DEFER** | The Docs Agent is always invoked by the Orchestrator, which validates routing before delegation. Step 0 would add consistency but zero practical value. |
| B9 | R4-M08 | `retro-spec.agent.md` | Step 0a scans for `target_agent: "7. Retro-Spec"` schema, none exists — always a no-op | **FIX** | Remove the scan. Replace with: "Retro-Spec is user-invoked. No inbound handoff schema validation is performed." |
| B10 | R4-M15 | `coder-complete-to-orchestrator.schema.yaml` | Schema exists but Coder never references it; `lane_confirmation` field unused | **ACCEPT** | Document as Orchestrator-facing documentation schema (same as B7 group). |
| B11 | R4-M03 | `spec-architect-to-orchestrator.schema.yaml` | `field: "status"` (lowercase) should be `"Status"` (uppercase); uses `expected:` not `value:` | **FIX** | Two-word fix: `status → Status`, `expected → value`. |
| B12 | R4-M05 | `orchestrator.agent.md` | State Verification Protocol doesn't check `docs_completed` — could re-invoke Docs Agent | **FIX** | Add `docs_completed` check to the protocol: "Also read `docs_completed` from each WP frontmatter to determine documentation status." |

**Effort**: 8 FIX (mostly 1–3 line edits), 2 ACCEPT (add comments), 2 DEFER.

---

## Group C: WP Frontmatter Formalization

*This is one structural gap: no single document defines the WP frontmatter schema. The fix is one new section in the Planner + updates to `base-handoff.schema.yaml`.*

| # | ID | File(s) | Issue | Disposition | Fix |
|---|---|---|---|---|---|
| C1 | R4-S05 | `planner.agent.md` | WP template shows only markdown body, no YAML frontmatter | **FIX** | Add a complete YAML frontmatter template to the Planner's WP file specification. |
| C2 | R4-S06 | `planner.agent.md` | `target_framework` never written to WP frontmatter — Coder reads it with no fallback | **FIX** | Include `target_framework` in the frontmatter template (C1). Add fallback in Coder: if absent, use empty string. |
| C3 | R4-M02 | `base-handoff.schema.yaml` | `wp_frontmatter_fields` documents only 2 of 8+ fields | **FIX** | Add: `lane`, `review_status`, `depends_on`, `target_language`, `target_framework`, `docs_scope`, `coverage_code`, `coverage_branch`, `Spec`. |
| C4 | new-P10 | `planner.agent.md` / `coder.agent.md` | `review_status` field never initialized by Planner; Coder reads it in rework mode | **ACCEPT** | The field is undefined on fresh WPs and only set by the Coder on rework. This is fine — the Coder already handles the absent case as "not in rework." Document in C1 template: `review_status: # set by Coder on rework, absent on fresh WPs`. |

**Effort**: One new frontmatter template section (C1 covers C2), one schema update (C3). C4 is documentation in the template.

---

## Group D: Contract-to-Agent Template Sync

*Every skill contract has drifted from its coordinator agent's actual dispatch template. These are documentation updates to the contract files — they don't change runtime behavior but prevent skill authors from building against the wrong interface.*

| # | ID | File(s) | Issue | Disposition | Fix |
|---|---|---|---|---|---|
| D1 | R4-C04 | `code-implementation/SKILL.md`, `CODER-SKILL-CONTRACT.md` | Skill designed for WP-batch; agent dispatches per-task. Dead dependency-graph logic in skill. | **FIX** | Update `code-implementation/SKILL.md` to document single-task dispatch: one invocation = one task. Remove internal dependency-graph logic description. Update CODER-SKILL-CONTRACT Section 5 to say "dispatched once per task." |
| D2 | R4-S02 | `coder.agent.md`, `code-debug/SKILL.md` | Debug dispatch template missing 5 of 11 inputs (`wp_path`, `patterns`, `target_language`, `target_framework`, `task_list`) | **FIX** | Add the 5 missing inputs to the Step 7 debug dispatch template. Add a substitution values table. |
| D3 | R4-M13 | `CODER-SKILL-CONTRACT.md` | Section 4 template shows 7 items; agent sends 11–12. Missing: `shared_contracts_dir`, `artifact_summary`, `research_context`, `dependency_source_summary`, `prior_task_files`. | **FIX** | Update Section 4 to show both WP-level (11 items) and single-task (12 items) templates. |
| D4 | R4-S11 | `DOC-SKILL-CONTRACT.md`, `docs-agent.agent.md` | Contract specifies 6 inputs; agent sends 7 (`contracts_dir` as separate item) | **FIX** | Add `contracts_dir` as Input #7 in DOC-SKILL-CONTRACT. |
| D5 | R4-M16 | `DOC-SKILL-CONTRACT.md`, `docs-agent.agent.md` | `doc-inline-code` `docs_dir` exception documented in contract but never implemented in agent | **FIX** | Remove the exception from DOC-SKILL-CONTRACT. The universal template is correct — receiving an unused `docs_dir` causes no harm. |
| D6 | R4-M12 | `REVIEW-SKILL-CONTRACT.md` | Section 5 shows flat list of 8 skills; agent implements 3-batch structure with skip logic | **FIX** | Document the batch structure and re-review scoping in Section 5. |
| D7 | R4-M14 | `RETRO-SKILL-CONTRACT.md`, `retro-spec.agent.md` | Contract Section 7 has single table; agent excludes `retro-architecture` from module-level dispatch | **FIX** | Add project-level vs module-level columns to Section 7. |
| D8 | new-R01 | `REVIEW-SKILL-CONTRACT.md` | Input #4 `contracts_dir` listed as mandatory but absent from dispatch template | **FIX** | Either add `contracts_dir` to the dispatch template, or change input #4 from "mandatory" to "available via `spec_path` inference." |
| D9 | new-R04 | `REVIEW-SKILL-CONTRACT.md` | No documentation of pattern file curation system (Step 14 of review-coordinator) | **DEFER** | Nice-to-have. Pattern curation is a coordinator concern, not a skill concern. Add a brief note referencing review-coordinator Step 14. |
| D10 | new-RE01 | `RETRO-SKILL-CONTRACT.md` | Module dispatch uses dual accumulator paths (project + module) but contract documents only one `accumulator_path` | **FIX** | Add `project_spec_path` as a distinct input for module-level dispatches. |
| D11 | new-RE02 | `RETRO-SKILL-CONTRACT.md` | `retro-discovery` exception uses `source_path` but agent dispatches `codebase_path`; agent hardcodes output path but contract says `output_path` variable | **FIX** | Align naming: `source_path → codebase_path`, document output as hardcoded path. |
| D12 | new-RS1 | `retro-spec.agent.md`, `retro-assembly/SKILL.md` | Assembly dispatch omits `all_project_specs`, `source_path`, `project_name` — 3 of 9 inputs | **FIX** | Add the 3 missing inputs to the Step 4 assembly dispatch template. |
| D13 | new-SC02 | `SPEC-SKILL-CONTRACT.md` | Section 6 lists skill→section mappings but omits companion artifact assignments (which skills produce which `.ext` files) | **DEFER** | Nice-to-have for skill authors. Spec Architect Step 7b documents this fully. |
| D14 | new-PP45 | `PLAN-SKILL-CONTRACT.md`, `planner.agent.md` | `phase` input (#9) listed in contract but never passed in either dispatch template | **FIX** | Add `phase: 1` or `phase: 2` to the respective dispatch templates. |
| D15 | new-PP23 | `PLAN-SKILL-CONTRACT.md`, `planner.agent.md` | Phase 2 template missing `patterns` (input #8) and `research_summary` (input #6) | **FIX** | Add both to the Phase 2 dispatch template. |
| D16 | new-PP06 | `PLAN-SKILL-CONTRACT.md`, `planner.agent.md` | Phase 1 template missing `contracts_dir` (input #3) | **FIX** | Add `contracts_dir` to Phase 1 dispatch template. |
| D17 | new-D04 | `DOC-SKILL-CONTRACT.md` | `docs_scope` WP frontmatter filter (docs-agent Step 5b) completely absent from contract | **DEFER** | Add a note in contract: "The coordinator may filter skill dispatch based on WP `docs_scope` frontmatter." |
| D18 | new-RE04 | `RETRO-SKILL-CONTRACT.md` | `retro-test-analysis` shown as inline phase 6; agent dispatches it as a separate Step 3 post-loop | **DEFER** | Add a note: "retro-test-analysis runs after extraction loop, not inline." |

**Effort**: 14 FIX (contract text edits), 4 DEFER (documentation improvements).

---

## Group E: Documentation, Cosmetics, and Edge Cases

*These have no runtime impact. Fix at your discretion or acknowledge and move on.*

| # | ID | File(s) | Issue | Disposition | Fix |
|---|---|---|---|---|---|
| E1 | R4-C01 | `orchestrator.agent.md` | Decision Table row citations off-by-two in Step 5 | **ACCEPT** | LLMs don't count table rows by number — they read the condition text inline. The citations are misleading comments, not executable routing. Add a comment: "Row numbers are approximate references, not routing keys." |
| E2 | R4-N01 | `orchestrator-reference.md` | Retry log `{retry_count + 1}` off-by-one — always prints "attempt 2 of 2" | **FIX** | One-word fix: `{retry_count + 1}` → `{retry_count}`. |
| E3 | R4-N02 | `orchestrator.agent.md` | Step 7 references `replace_string_in_file` tool — not in agent toolset | **FIX** | Replace with `edit/editFiles`. |
| E4 | R4-N03 | `orchestrator.agent.md` | Coder prompt template says "Dependency {dep_wp}" for WPs with no deps | **FIX** | Add: "Include dependency sentence only when `depends_on` is non-empty." |
| E5 | R4-N04 | `orchestrator.agent.md` | No backward state transitions (ideation→idle, etc.) on failure | **ACCEPT** | The pipeline stays at the current stage during retries. The retry/escalation protocol handles failures without transitioning backward. Add a note: "On agent failure, `pipeline_stage` remains unchanged until the escalation resolves." |
| E6 | R4-M07 | `orchestrator.agent.md` | No exit from `pipeline_stage: complete` — re-running after completion is undefined | **FIX** | Add transition: `complete → idle` (triggered by Orchestrator restart with `pipeline_stage: complete`). |
| E7 | R4-N05 | `planner.agent.md` | Step 4a references "Explore agent" — not a pipeline agent | **FIX** | Change to use direct search tool calls or document "Explore" as the VS Code Copilot chat participant name. |
| E8 | R4-N06 | All 5 `*-SKILL-CONTRACT.md` | All reference `.sdd/specs/NNN-*.spec.md` that don't exist in the repo | **ACCEPT** | These specs were used during development and are expected to be absent from the published repo. The FR references are provenance markers, not runtime dependencies. Add header note: "FR references are from the original design specs, not shipped with this repository." |
| E9 | R4-N07 | `enums.yaml` | `review_status: approved` never set by any agent — dead value | **FIX** | Remove `approved` from the enum. |
| E10 | R4-N08 | `ideation.agent.md` | Em dash `—` in `<brief_template>` title violates the agent's own encoding rule | **FIX** | Change `—` to `-` in the template. |
| E11 | R4-N09 | `enums.yaml` / `review-coordinator.agent.md` | Verdict casing: agent uses Title Case, enum uses snake_case. No normalization documented. | **FIX** | Add a comment block to `enums.yaml`: "Enum values are snake_case canonical forms. Agents use Title Case in prose output. The Orchestrator normalizes before state comparison." |
| E12 | R4-N10 | `ideation.agent.md` | Failure-branch cleanup uses wildcard `Remove-Item .sdd/research-*.md` | **ACCEPT** | The concurrent-agent scenario (Ideation + Brainstorming writing simultaneously) is theoretical — an interactive user runs one at a time. Risk is negligible. |
| E13 | new-I3 | `ideation.agent.md` | No mechanism to enforce 5-round loop limit — counter relies on implicit self-counting | **ACCEPT** | LLMs can count their own exchanges. No enforcement mechanism is needed beyond the stated rule. |
| E14 | new-B3 | `brainstorming.agent.md` | Session tracking lacks "Round Count" category | **DEFER** | Add "Current Round: N/10" to session tracking template. Minor usability improvement. |
| E15 | new-SA2 | `spec-architect.agent.md` | Handoff prompts have unfilled `<spec_path>`, `<artifacts_dir>` template tokens for `send: false` handoffs | **ACCEPT** | `send: false` means the user manually triggers the button and sees the prompt. Template tokens are visual cues for the user — not machine-parsed. |
| E16 | new-SA3 | `spec-architect.agent.md` | Spec artifacts `git add /*` uses glob, not explicit file list (violates "always list files explicitly" rule) | **DEFER** | The rule exists to prevent accidental staging. Globbing within a known artifacts subdirectory is safe. |
| E17 | new-C02 | `coder.agent.md` | Sub-step ordering: Step 2d runs before 2b, wasting dependency context work in rework mode | **DEFER** | Performance optimization, not a correctness issue. |
| E18 | new-C07 | `coder.agent.md` | One occurrence uses `askQuestions` instead of `vscode_askQuestions` | **FIX** | Fix the tool name. |
| E19 | new-C10 | `coder.agent.md` | Coverage remediation re-dispatches `code-integration-tests` (ineffective for line/branch coverage) | **ACCEPT** | The instruction says "re-dispatch the test skills" — the agent can choose to skip integration tests if coverage gaps are unit-test-addressable. |
| E20 | new-C12 | `coder.agent.md` | `git diff` without scope in Step 6b.2 — shows all unstaged changes, not just WP-related | **DEFER** | Minor. Agent can interpret relevant changes from context. |
| E21 | new-RC4 | `review-coordinator.agent.md` | No fallback when spec has no "Source brief" field (retro-spec specs don't have one) | **FIX** | Add: "If `Source brief` field is absent, skip brief reading and continue." |
| E22 | new-RC5 | `review-coordinator.agent.md` | Orchestrator-sourced invocations not schema-validated (only `coder-to-reviewer` handled) | **ACCEPT** | Same reasoning as B7: Orchestrator controls routing. Step 0 validates the primary path. |
| E23 | new-RC6 | `review-coordinator.agent.md` | `review-spec-completeness` in pattern domain mapping table is unreachable dead code | **FIX** | Remove the `review-spec-completeness → spec-patterns.md` row from Step 14a. |
| E24 | new-DA5 | `docs-agent.agent.md` | `docs_completed: true` set even on no-op runs — Orchestrator can't distinguish docs-generated from nothing-changed | **ACCEPT** | The Orchestrator only cares "was the Docs Agent invoked?" not "was output produced?" No-op is fine — the docs are simply already current. |
| E25 | new-RS4 | `retro-spec.agent.md` | No patterns consumption step — anti-patterns never reach extraction skills | **DEFER** | Retro-spec is the newest agent. Pattern files don't exist yet for retro extractions. When they do, add a read step. |
| E26 | new-RS5 | `retro-spec.agent.md` | Option B promotes specs to "Validated" by file copy without review | **ACCEPT** | Option B is explicitly documented as the fast path. Option A (recommended) goes through Spec Architect. The user makes the choice. |
| E27 | new-P11 | `planner.agent.md` | Commit policy claims per-WP commits but workflow does single batch commit at Step 12 | **ACCEPT** | The batch commit at Step 12 is actually correct — you don't want partial WP sets committed. The policy description is aspirational; the workflow is practical. |
| E28 | new-SC03 | `SPEC-SKILL-CONTRACT.md` | Companion artifact comment syntax table missing Rust, Java, C# (present in RETRO contract) | **DEFER** | Add languages when a project targeting them is first built. |
| E29 | new-SA4 | `spec-architect.agent.md` | Skill failure halts mid-spec with no "INCOMPLETE" status written | **DEFER** | The halt + error escalation is sufficient. Adding an INCOMPLETE status marker is useful but not blocking. |
| E30 | new-Sch9 | `docs-agent-to-coder.schema.yaml`, `docs-agent-to-reviewer.schema.yaml` | `value_one_of: [true, false]` accepts any boolean — no-op validation | **ACCEPT** | Schemas exist for documentation (B7 reasoning). The validation is structurally correct even if tautological. |
| E31 | new-Sch5 | `ideation-to-orchestrator.schema.yaml` | Artifact path `{slug}` doesn't model `{NNN}-{name}` naming; missing `version_history` | **DEFER** | Schema is documentation-only (Orchestrator has no Step 0). Pattern is informational. |
| E32 | new-C05 | `coder.agent.md` | Rework prompt (Step 6b.3) has no substitution values table — FB-XX list format undefined | **FIX** | Add a brief substitution table defining the format: `<FB-XX list>: FB-01 description (file:line)` etc. |

**Effort**: 10 FIX (1–3 line edits each), 13 ACCEPT, 9 DEFER.

---

## Summary by Disposition

| Disposition | Count | Groups |
|---|---|---|
| **FIX** | 44 | A: 12, B: 8, C: 3, D: 14, E: 10 |
| **ACCEPT** | 17 | B: 2, C: 1, E: 13, Group-wide: 1 |
| **DEFER** | 13 | B: 2, D: 4, E: 9 |
| **Total** | **74** | |

---

## Recommended Execution Order

| Pass | Group | Description | Estimated edits |
|---|---|---|---|
| **1** | **A** (Agent Logic) | Fix the 12 internal contradictions that could cause wrong behavior | 12 edits across 5 agent files |
| **2** | **C** (WP Frontmatter) | Create the frontmatter template + update `base-handoff.schema.yaml` | 2 files |
| **3** | **B** (Handoff/Schema) | Fix 8 handoff prompts and schema fields | 8 edits across 6 files |
| **4** | **D** (Contract Sync) | Update 14 contract/SKILL.md entries to match actual agent dispatch | 14 edits across ~10 files |
| **5** | **E** (Cleanup) | Apply 10 small fixes; acknowledge 13 ACCEPT items | 10 edits |

After passes 1–3, the pipeline has no issues that affect runtime behavior on any standard path.
Passes 4–5 are documentation quality improvements.

---

## Closure Criteria

- [ ] All 44 FIX items resolved
- [ ] All 17 ACCEPT items reviewed (add inline comments where noted)
- [ ] 13 DEFER items logged in a backlog (optional)
- [ ] No Round 5 adversarial review needed
