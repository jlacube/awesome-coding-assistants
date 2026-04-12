# Adversarial Review — Round 4

**Date**: 2026-04-12  
**Scope**: `.github/agents/`, `.github/skills/`, `.github/schemas/`, `.github/prompts/`  
**Reviewer**: Automated adversarial audit  
**Baseline**: ADVERSARIAL-REVIEW-R3.md (11 of 13 R3 issues fully fixed, 2 partially fixed — see Appendix A)

---

## Executive Summary

All 13 issues from Round 3 have been resolved (11 fully, 2 partially — the partials are re-tracked below). This round discovered **5 critical**, **12 significant**, **16 moderate**, and **10 minor** new issues. The most impactful systemic findings are: (1) all Decision Table row citations in the Orchestrator are off-by-two, routing pipeline-complete to the blocked-WP handler; (2) four `send: true` handoff buttons will halt the pipeline on missing required schema fields; (3) the `code-implementation` skill is designed for WP-batch dispatch but receives single-task dispatch; and (4) eight handoff schemas are orphaned — no target agent validates them.

---

## Severity Definitions

| Severity | Meaning |
|----------|---------|
| **Critical** | Pipeline will halt, malfunction, or produce wrong output on standard paths |
| **Significant** | Incorrect cross-references, missing constraints, or broken dispatch that could mislead agents or skip validation |
| **Moderate** | Missing validation, undefined fields, stale contracts, or inconsistencies with limited runtime impact |
| **Minor** | Documentation staleness, cosmetic issues, dead code, or theoretical edge cases |

---

## New Findings

### C-01 [Critical] — Orchestrator Decision Table row citations are all wrong

**File**: `orchestrator.agent.md` Step 5

Step 5 references Decision Table rows by number in two priority blocks. All four citations are off by one or two:

| Step 5 location | Claims row | Actual row | Actual row meaning |
|---|---|---|---|
| Priority 1, retry | row 11 | **row 12** | Agent failure, retry_count < 2 |
| Priority 1, escalate | row 12 | **row 13** | Agent failure, retry_count >= 2 |
| Priority 3, MVP ask | row 10 | **row 11** | All MVP WPs done, non-MVP remain |
| Priority 3, complete | **row 9** | **row 10** | All WPs done — pipeline complete |

The most dangerous: Priority 3 item 2 cites **row 9** for "pipeline complete." Row 9 is actually `"WP has lane: blocked → Escalate to user."` An LLM following this citation literally would route pipeline completion into the blocked-WP escalation path.

**Fix**: Update all four row citations in Step 5 to match the actual Decision Table numbering.

---

### C-02 [Critical] — Ideation `send: true` handoff omits required `brief_path`

**File**: `ideation.agent.md` frontmatter handoff to Spec Architect

The `send: true` handoff prompt is `"Develop the ideation brief into a full specification"` — with no `brief_path` field.

`ideation-to-spec.schema.yaml` marks `brief_path` as `required: true`. Spec Architect Step 0 sub-step 5 will halt: *"Missing required context field: brief_path."*

Every successful ideation run that auto-triggers Spec Architect via the handoff button will fail at schema validation.

**Fix**: Add `brief_path: .sdd/ideas/<NNN>-<idea-name>.md` to the handoff prompt, or populate it dynamically from the agent's output.

---

### C-03 [Critical] — Brainstorming both `send: true` handoffs omit required `brief_path`

**File**: `brainstorming.agent.md` frontmatter handoffs

Both handoff buttons ("Develop into Specification" and "Continue as Standard Ideation") have `send: true` but neither prompt includes `brief_path`. `brainstorming-to-spec.schema.yaml` marks `brief_path` as `required: true`.

Same consequence as C-02: the auto-triggered Spec Architect will halt on schema validation.

**Fix**: Add `brief_path` to both handoff prompts.

---

### C-04 [Critical] — `code-implementation` SKILL.md vs Coder dispatch architecture mismatch

**Files**: `code-implementation/SKILL.md`, `coder.agent.md` Step 6

The SKILL.md Input Contract #9 is `task_list` and Step 1 says: *"One code-implementation invocation handles all tasks in one WP. Process tasks sequentially in dependency order."*

The Coder agent dispatches `code-implementation` **once per task** using a single-task template. Only one task's data is sent per invocation. The skill's internal dependency-graph logic is entirely moot — it never receives more than one task.

Additionally, `CODER-SKILL-CONTRACT.md` Section 5 says `code-implementation` is dispatched "once per WP," contradicting the agent's per-task loop.

**Impact**: The skill's dependency-ordering and multi-task pipeline logic is dead code. If a skill author implements it faithfully, the WP-batch architecture will never execute.

**Fix**: Either update the SKILL.md to describe single-task dispatch semantics, or update the Coder agent to dispatch once per WP. Update CODER-SKILL-CONTRACT.md to match whichever is chosen.

---

### C-05 [Critical] — `review_cycles` escalation threshold: `>= 2` vs `>= 3` — behavioral contradiction

**Files**: `orchestrator.agent.md` (three locations), `orchestrator-reference.md`

The Orchestrator defines the review stall threshold in three places, all saying `>= 2`:
- Step 5, Priority 2: `"review_cycles >= 2" -- escalate`
- Step 8e: `"If WP review_cycles >= 2, halt and escalate"`
- Failure Summary Table: `"WP review_cycles >= 2 at for_review | Escalate"`

The `orchestrator-reference.md` says `>= 3`. The Failure Summary Table itself is internally inconsistent: one row says `review_cycles >= 2` and a later row says `"Same WP fails review 3 times."` Step 8e acknowledges the Review Coordinator's own threshold is `>= 3` and says the Orchestrator should fire "before" it — confirming `>= 2` is intentional. But the reference file contradicts this.

**Impact**: An implementation following the reference allows a third review attempt; one following the agent halts after the second. This is a runtime behavioral difference.

**Fix**: Align `orchestrator-reference.md` to `>= 2` (matching the agent's three consistent uses), or update the agent to `>= 3` if three attempts is the intended design. Update the Failure Summary Table's "3 times" row to match.

---

### S-01 [Significant] — Coder schema validation has no branch for `lane: doing`

**File**: `coder.agent.md` Step 0

Step 0 defines schema selection for exactly two cases:
- `lane: planned` → `planner-to-coder.schema.yaml`
- `lane: to_do` → `reviewer-to-coder.schema.yaml`

But Step 2b recognizes `lane: doing` as valid Standard Mode. A WP left at `lane: doing` (Coder interrupted mid-implementation) fails both schemas. The agent halts with a schema error on every re-invocation of an in-progress WP.

**Fix**: Add a third branch to Step 0: `lane: doing` → skip schema validation (resume context) or use a dedicated resume schema.

---

### S-02 [Significant] — Coder debug dispatch missing 5 of 11 required inputs

**Files**: `coder.agent.md` Step 7, `code-debug/SKILL.md`

The Step 7 debug dispatch template provides only 6 items. Five SKILL.md-defined inputs are missing: `wp_path`, `patterns`, `target_language`, `target_framework`, `task_list`. Every other dispatch template in the Coder includes these.

Additionally, the debug template has no substitution values table (unlike Step 6 templates), making `<skill_path>` and other placeholders ambiguous.

**Fix**: Add the missing 5 inputs and a substitution values table to the Step 7 debug dispatch template.

---

### S-03 [Significant] — Planner writes to `.sdd/plans/` before the directory exists

**File**: `planner.agent.md` Steps 2 vs 6

Step 2 dispatches `review-spec-completeness` with output path `.sdd/plans/spec-completeness-report.md`. Step 6 is where `.sdd/plans/` is first created. Step 2 runs before Step 6 — the write will fail on a fresh project.

**Fix**: Either create `.sdd/plans/` in Step 2 before dispatching the skill, write the report to `.sdd/` instead, or move the directory creation before Step 2.

---

### S-04 [Significant] — Planner schema hardcoded to `spec-to-planner.schema.yaml`, ignores retro path

**File**: `planner.agent.md` Step 0

Step 0 unconditionally validates against `spec-to-planner.schema.yaml`. `retro-spec-to-planner.schema.yaml` exists with a distinct `source_agent` ("7. Retro-Spec") and additional context field `legacy_path`. When Retro-Spec hands off to the Planner, the wrong schema is validated. The `legacy_path` field is silently ignored.

**Fix**: Add a conditional branch: if `source_agent` is `"7. Retro-Spec"`, validate against `retro-spec-to-planner.schema.yaml`.

---

### S-05 [Significant] — Planner WP template omits YAML frontmatter

**File**: `planner.agent.md` WP file template section

The WP template in the Planner shows only the markdown body (`## Objective`, `## Tasks`, `## Activity Log`). The YAML frontmatter block that the Coder requires is entirely absent. Missing fields the Coder reads from frontmatter: `lane`, `target_language`, `target_framework`, `coverage_code`, `coverage_branch`, `review_status`, `depends_on`, `Spec`.

The `plan-decomposition` skill must invent the frontmatter schema from context since no template provides it.

**Fix**: Add a complete YAML frontmatter template with all fields to the Planner's WP file specification.

---

### S-06 [Significant] — `target_framework` never written to WP frontmatter by Planner

**Files**: `planner.agent.md`, `coder.agent.md` Step 2

The Planner's Phase 1 dispatch template passes `target_language` but not `target_framework`. No instruction tells plan skills to write `target_framework` into WP frontmatter.

The Coder reads `target_framework` from WP frontmatter at Step 2 item 6 with no fallback for missing values (unlike `target_language`, which falls back to the spec). Every WP created by the Planner will pass to the Coder with an undefined `target_framework`.

**Fix**: Add `target_framework` to the Planner's dispatch template and WP frontmatter template.

---

### S-07 [Significant] — 8 handoff schemas are orphaned — target agent never validates them

**Files**: Various schemas in `.github/schemas/`

These schemas exist but no receiving agent reads or validates against them:

| Schema | Why orphaned |
|---|---|
| `brainstorming-to-spec.schema.yaml` | Spec Architect Step 0 uses `ideation-to-spec` for both Ideation and Brainstorming |
| `coder-to-spec.schema.yaml` | Spec Architect Step 0 has no "from Coder" branch |
| `retro-spec-to-spec.schema.yaml` | Spec Architect Step 0 has no "from Retro-Spec" branch |
| `coder-to-planner.schema.yaml` | Planner hardcodes `spec-to-planner.schema.yaml` |
| `retro-spec-to-planner.schema.yaml` | Same — Planner hardcodes one schema |
| `docs-agent-to-coder.schema.yaml` | Coder Step 0 only handles Planner and Reviewer sources |
| `docs-agent-to-reviewer.schema.yaml` | Review Coordinator hardcodes `coder-to-reviewer` |
| `spec-architect-to-ideation.schema.yaml` | Ideation has no Step 0 schema validation at all |

These schemas were created to fix R3's M-04, but the agent files were not updated to reference them. The schema layer promises validation it cannot deliver.

**Fix**: For each orphaned schema, either add a Step 0 branch in the target agent to handle the source, or remove the schema and document the exemption.

---

### S-08 [Significant] — Review Coordinator promotes spec status from "Draft" → "Approved" bypassing validation gate

**File**: `review-coordinator.agent.md` Step 13a

Step 13a updates the spec `Status` field from `"Draft or Validated"` to `"Approved"` when all WPs complete. A spec in "Draft" (never validated by humans) would be auto-promoted to "Approved."

`retro-spec-to-planner.schema.yaml` requires `Status: Validated` before planning, establishing "Validated" as a mandatory checkpoint — this bypasses it.

**Fix**: Only allow promotion from `Validated` → `Approved`. Emit a warning if `Status: Draft` is found.

---

### S-09 [Significant] — Review Coordinator commit retry policy contradicts between Steps 14g and 15

**File**: `review-coordinator.agent.md` Steps 14g vs 15

Step 14g: *"If the commit fails, retry once. If it fails again, report the error and continue with the review commit (Step 15)."*

Step 15: *"If the git commit fails, report the error to the user and halt. Do not retry."*

Direct contradiction on retry policy (retry once vs. never) and failure behavior (continue vs. halt).

**Fix**: Unify the commit failure policy across both steps.

---

### S-10 [Significant] — Docs Agent has no Step 0 schema validation

**File**: `docs-agent.agent.md`

Spec Architect, Review Coordinator, Coder, and Retro-Spec all have Step 0 schema validation. The Docs Agent begins at Step 1 with only manual checks. No incoming handoff schema is validated. `docs-agent-to-orchestrator.schema.yaml` exists but is never referenced by the agent.

**Fix**: Add Step 0 schema validation consistent with other coordinator agents.

---

### S-11 [Significant] — Docs Agent dispatch sends 7 inputs; DOC-SKILL-CONTRACT defines only 6

**Files**: `docs-agent.agent.md` Step 6a, `DOC-SKILL-CONTRACT.md`

The dispatch template includes `contracts_dir` as a separate item 4, creating 7 dispatch items. The contract folds `contracts_dir` into the `spec_path` description (Input #3: "Path to the spec file; includes contract files directory"), producing only 6 inputs.

Skills built against the contract expect `contracts_dir` information embedded in `spec_path`. Skills built against the dispatch prompt receive it as a separate item. This creates ambiguity for skill authors.

**Fix**: Add `contracts_dir` as Input #7 in DOC-SKILL-CONTRACT, or remove it from the dispatch template if skills can derive it from `spec_path`.

---

### S-12 [Significant] — Coder `lane: to_do` with no FB-XX items is unhandled

**File**: `coder.agent.md` Step 2b

Step 2b defines two modes:
- Rework: `lane: to_do` AND FB-XX items exist
- Standard: `lane: planned`, `lane: doing`, or no feedback

If the Reviewer sets `lane: to_do` but omits FB-XX findings, the Coder enters Standard Mode and re-runs the full 5-skill pipeline on an already-implemented WP, risking overwrites.

**Fix**: Add a guard: if `lane: to_do` and no FB-XX items exist, halt with an error message.

---

### M-01 [Moderate] — `reviewer-to-coder.schema.yaml` missing `spec_path` and `contracts_dir`

**Files**: `reviewer-to-coder.schema.yaml`, `coder.agent.md` Step 6b

The schema's `context_fields` defines only `wp_path` (required) and `review_report_path` (optional). Missing: `spec_path` (required by all Coder dispatch templates for `<spec_refs>`) and `contracts_dir` (required by skill dispatches). `planner-to-coder.schema.yaml` correctly includes both fields.

**Fix**: Add `spec_path` and `contracts_dir` as required context fields to `reviewer-to-coder.schema.yaml`.

---

### M-02 [Moderate] — `base-handoff.schema.yaml` documents only 2 of 8+ WP frontmatter fields

**File**: `base-handoff.schema.yaml` `wp_frontmatter_fields`

The section defines `review_cycles` and `docs_completed` only. Actively-used fields missing: `lane` (the most critical pipeline field), `review_status`, `depends_on`, `target_language`, `target_framework`, `docs_scope`.

**Fix**: Add all actively-used WP frontmatter fields to the schema.

---

### M-03 [Moderate] — `spec-architect-to-orchestrator.schema.yaml` case mismatch on `status` field

**File**: `spec-architect-to-orchestrator.schema.yaml`

Validation rule uses `field: "status"` (lowercase). Spec frontmatter uses `Status:` (uppercase). Compare with `spec-to-planner.schema.yaml` which correctly uses `field: "Status"` (uppercase).

Also uses `expected:` keyword instead of the `value:` keyword used by all other schemas.

**Fix**: Change `field: "status"` to `field: "Status"` and `expected:` to `value:`.

---

### M-04 [Moderate] — Orchestrator corrupted state recovery omits `lane: to_do` and `lane: blocked`

**File**: `orchestrator-reference.md` Section 2

The reconstruction logic for `pipeline_stage` covers `doing/planned → implementation`, `for_review → review`, `all done → check docs`. It omits:
- `lane: to_do` (WP needing rework) — falls through silently
- `lane: blocked` — similarly uncovered
- Mixed lanes (e.g., one WP `for_review`, another `doing`) — no priority rule
- Empty `.sdd/plans/` directory — undefined behavior

**Fix**: Add `to_do → implementation`, `blocked → same (escalate)`, and define mixed-lane priority.

---

### M-05 [Moderate] — Orchestrator State Verification Protocol doesn't check `docs_completed`

**File**: `orchestrator.agent.md` State Verification Protocol

The startup cross-check example: *"proceed to documentation for WP03 or the next WP if docs are already done."* But it never instructs checking `docs_completed` to determine "already done." This check is only in Step 5. A stale restart could re-invoke the Docs Agent on an already-documented WP.

**Fix**: Add `docs_completed` to the State Verification Protocol checklist.

---

### M-06 [Moderate] — Decision Table row 11 `State After: idle` is wrong for both user responses

**File**: `orchestrator.agent.md` Decision Table row 11

Row 11: *"All MVP WPs done, non-MVP remain | Ask user | idle."* But:
- User says "yes, continue" → state should be `implementation`, not `idle`
- User says "no, halt" → state should be `complete`, not `idle`

`idle` means "nothing exists yet" — the Orchestrator would restart from scratch.

**Fix**: Remove the fixed `idle` and add conditional transitions based on user response.

---

### M-07 [Moderate] — No exit transition from `pipeline_stage: complete`

**File**: `orchestrator.agent.md` State Transition Table

The table's 13 rows have no transition FROM `complete`. If a user re-runs the Orchestrator after completion (e.g., after manually adding a new WP), the transition table has no valid path from `complete` to `implementation`. The Orchestrator must either violate the "verify transition is valid" rule or halt.

**Fix**: Add `complete → idle` transition (triggered by Orchestrator restart), or add `complete → implementation` (triggered by new WP detected).

---

### M-08 [Moderate] — No schema exists with `target_agent: "7. Retro-Spec"`

**File**: `retro-spec.agent.md` Step 0a, all schemas

Step 0a searches schemas for `target_agent: "7. Retro-Spec"`. No such schema exists. `orchestrator-handoff.schema.yaml` has `target_agent: "*"` but the agent doesn't match wildcards. The agent always falls to "skip schema validation" — making Step 0a a no-op.

**Fix**: Either create a schema with `target_agent: "7. Retro-Spec"`, add wildcard matching to Step 0a, or remove the scan and document that retro-spec skips inbound validation.

---

### M-09 [Moderate] — Brainstorming rules vs readiness criteria contradict on early wrap-up

**File**: `brainstorming.agent.md`

Rule: *"if the user asks to wrap up early before round 10, inform them that at least 10 rounds are required...and continue exploring"* — unconditional refusal.

Readiness Criteria: *"At least 10 rounds of Q&A have been completed (or user explicitly waives this)"* — allows waiver.

**Fix**: Reconcile — either allow waiver in rules or remove waiver from readiness criteria.

---

### M-10 [Moderate] — Spec Architect Step 0 uses wrong schema for Brainstorming handoffs

**File**: `spec-architect.agent.md` Step 0

Step 0: *"If the handoff comes from Ideation/Brainstorming: read ideation-to-spec.schema.yaml."* But `brainstorming-to-spec.schema.yaml` exists as a distinct schema. The schema is always bypassed; `source_agent: "1. Ideation"` is wrong when the source is actually `"1.1. Brainstorming"`.

**Fix**: Add a separate branch for Brainstorming handoffs using `brainstorming-to-spec.schema.yaml`.

---

### M-11 [Moderate] — Review Coordinator pattern files created without `patterns_version` frontmatter

**File**: `review-coordinator.agent.md` Step 14b

Step 14b creates new pattern files with no YAML frontmatter. `patterns_version` is only added when a pattern is first written (Step 14f). Until then, agents that check `patterns_version` treat the absent field as 0, triggering an unnecessary reload on every skill dispatch.

**Fix**: Include `patterns_version: 1` in the initial pattern file template.

---

### M-12 [Moderate] — REVIEW-SKILL-CONTRACT omits batch dispatch structure

**File**: `REVIEW-SKILL-CONTRACT.md` Section 5

The contract presents a flat list of 8 skills. The Review Coordinator implements a 3-batch structure (Correctness, Safety, Polish) with batch-level skip logic on re-review. The contract has no concept of batches, making it impossible for skill authors to understand when their skill might be skipped.

**Fix**: Document the batch structure and re-review scoping logic in Section 5.

---

### M-13 [Moderate] — CODER-SKILL-CONTRACT Section 4 template is stale (7 items vs 11–12 in agent)

**File**: `CODER-SKILL-CONTRACT.md` Section 4

The contract's prompt template shows 7 items. The agent's actual dispatch template has 11–12 items. Missing from the contract: `shared_contracts_dir`, `artifact_summary`, `research_context`, `dependency_source_summary`, `prior_task_files`.

**Fix**: Update Section 4 to reflect the actual 11-item WP-level and 12-item single-task templates.

---

### M-14 [Moderate] — Retro-spec module-level dispatch silently drops `retro-architecture`

**File**: `retro-spec.agent.md` Step 2

Project-level dispatch includes `retro-architecture` as order 1. Module-level dispatch omits it entirely without documentation. RETRO-SKILL-CONTRACT Section 7 shows a single unified table with no project/module distinction.

**Fix**: Either document why `retro-architecture` is excluded from module-level dispatch, or add it. Update RETRO-SKILL-CONTRACT Section 7 to distinguish project vs module dispatch orders.

---

### M-15 [Moderate] — `coder-complete-to-orchestrator.schema.yaml` not referenced

**Files**: `coder-complete-to-orchestrator.schema.yaml`, `coder.agent.md` Step 9

The schema requires `lane_confirmation: "for_review"` as a structured field. The Coder's Step 9 completion report outputs `Lane: for_review` as prose text, not the schema's structured field. The schema is never referenced by the agent.

**Fix**: Either reference the schema in Step 9 and output the structured field, or remove the schema.

---

### M-16 [Moderate] — `doc-inline-code` `docs_dir` exception not honored by Docs Agent

**Files**: `DOC-SKILL-CONTRACT.md` exception note, `docs-agent.agent.md` Step 6a

The contract states `doc-inline-code` omits `docs_dir` (Input #5) because it writes to source files. The Docs Agent uses a single universal template for all skills, always including `.sdd/docs/`. The documented exception is never implemented.

**Fix**: Either customize the dispatch template for `doc-inline-code` to omit `docs_dir`, or update the contract to remove the exception.

---

### N-01 [Minor] — Orchestrator retry log message off-by-one

**File**: `orchestrator-reference.md`

Log message: `"Retrying {agent} for {wp} (attempt {retry_count + 1} of 2)"`. Since `retry_count` is already incremented before logging, the formula always produces "attempt 2 of 2" regardless of which attempt it is.

**Fix**: Use `{retry_count} of 2` (already-incremented value).

---

### N-02 [Minor] — Orchestrator `replace_string_in_file` tool reference invalid

**File**: `orchestrator.agent.md` Step 7

Step 7 says *"Write the updated state file back using replace_string_in_file."* This tool name doesn't appear in the agent's `tools:` frontmatter. The Orchestrator uses `edit/editFiles`.

**Fix**: Change the reference to `edit/editFiles`.

---

### N-03 [Minor] — Orchestrator Coder prompt template undefined for WPs with no dependencies

**File**: `orchestrator.agent.md` Step 6

The prompt template includes: *"Dependency {dep_wp} is lane=done (approved)."* For WPs with `depends_on: []` or absent `depends_on`, `{dep_wp}` is undefined.

**Fix**: Add a conditional: include the dependency sentence only when `depends_on` is non-empty.

---

### N-04 [Minor] — No backward state transitions on failure (ideation/specification/planning)

**File**: `orchestrator.agent.md` State Transition Table

No transitions exist for: `ideation → idle` (agent fails), `specification → ideation` (brief needs more detail), `planning → specification` (spec incomplete). The Orchestrator encountering a failure in `ideation` cannot legally transition anywhere.

**Fix**: Add backward transitions for failure cases, or rely on the retry/escalation protocol (if so, document that `pipeline_stage` stays constant during retries).

---

### N-05 [Minor] — Planner "Explore agent" reference is non-existent

**File**: `planner.agent.md` Step 4a

Step 4a: *"Dispatch a workspace research subagent using runSubagent with the Explore agent."* "Explore" is a VS Code Copilot chat mode, not a pipeline agent with an `.agent.md` file. This `runSubagent` call would fail.

**Fix**: Replace with the correct tool invocation for workspace exploration (e.g., direct `semantic_search` or equivalent).

---

### N-06 [Minor] — All 5 skill contracts reference non-existent `.sdd/specs/` files

**Files**: All `*-SKILL-CONTRACT.md` headers

| Contract | Referenced spec |
|---|---|
| CODER | `.sdd/specs/004-coder-v2.spec.md` |
| PLAN | `.sdd/specs/003-planner-v2.spec.md` |
| REVIEW | `.sdd/specs/006-review-coordinator.spec.md` |
| DOC | `.sdd/specs/007-docs-agent.spec.md` |
| SPEC | `.sdd/specs/002-spec-architect-v2.spec.md` |

The `.sdd/` directory does not exist in this repository. All FR citations in these contracts are unresolvable.

**Fix**: Either include the spec files in the repo, or remove the FR cross-references.

---

### N-07 [Minor] — `review_status: approved` in `enums.yaml` is an orphaned value

**File**: `enums.yaml`

The `approved` value is never set by any agent. Review Coordinator removes the `review_status` field entirely on approval (rather than setting it to `approved`). The value is dead.

**Fix**: Remove `approved` from the `review_status` enum, or document why it exists.

---

### N-08 [Minor] — Ideation em dash in `<brief_template>` title

**File**: `ideation.agent.md` brief template

Template line: `# [Idea Name] — Ideation Brief` uses U+2014 (em dash). The agent's own rule says: *"NEVER output em dashes...in brief files — use plain ASCII hyphens."* `brainstorming.agent.md` correctly uses `# [Idea Name] - Brainstorming Brief`.

**Fix**: Change `—` to `-` in the brief template.

---

### N-09 [Minor] — R3 S-02 partial: verdict casing normalization still undocumented

**Carried from R3 partial fix**

`enums.yaml` declares `approved_with_findings` (snake_case). The Review Coordinator and REVIEW-SKILL-CONTRACT use Title Case (`"Approved with Findings"`). No normalization rule or casing convention is documented.

**Fix**: Document the casing convention in `enums.yaml` or the Review Coordinator.

---

### N-10 [Minor] — R3 N-02 partial: ideation failure-branch cleanup still uses wildcard

**Carried from R3 partial fix**

Ideation success branch correctly uses specific file path cleanup. The failure branch still uses `Remove-Item .sdd/research-*.md`. This could delete brainstorming's in-progress research file if both agents run concurrently.

**Fix**: Use a specific filename in the failure branch (capture the filename before dispatch), or accept the risk with a documented note.

---

## Appendix A — R3 Fix Verification

| ID | Issue | Status |
|----|-------|--------|
| S-01 | SPEC-SKILL-CONTRACT Section 6 stale | **FIXED** — sections match Step 7b |
| S-02 | `verdict` enum missing `approved_with_findings` | **PARTIALLY FIXED** — enum value added; casing undocumented (→ N-09) |
| M-01 | `review_cycles` not in schema | **FIXED** — defined in `base-handoff.schema.yaml` |
| M-02 | retro-spec no Step 0 schema validation | **FIXED** — Step 0a added |
| M-03 | `retro-patterns.md` referenced but missing | **FIXED** — references removed |
| M-04 | 7 handoff schemas missing | **FIXED** — all created with proper content (but 6 orphaned — see S-07) |
| M-05 | `spec-api-design` missing `interfaces.<ext>` | **FIXED** — documented in SKILL.md |
| N-01 | Research cleanup not error-guarded | **FIXED** — both branches have cleanup |
| N-02 | Ideation wildcard cleanup | **PARTIALLY FIXED** — success uses specific path; failure still wildcard (→ N-10) |
| N-03 | `{NNN}` pattern in schema | **FIXED** — simplified to `*.md` glob |
| N-04 | `review-spec-completeness` not wired in Planner | **FIXED** — Step 2 dispatches it |
| N-05 | `semantic-commit` unused by agents | **FIXED** — documented as user-facing only |
| N-06 | `docs_completed` schema | **FIXED** — defined in `base-handoff.schema.yaml` |

---

## Appendix B — Systemic Themes

### Theme 1: Schema Layer Is Mostly Decorative

Of 25 handoff schemas, only 6 are actively validated by a receiving agent's Step 0. Eight are fully orphaned (S-07), six target the Orchestrator which has no Step 0 (Schema Issue 7), and others are used for the wrong source agent (M-10). The schema layer's promise of validated transitions is largely unfulfilled.

### Theme 2: Skill Contracts Diverge From Agent Dispatch Templates

Every skill contract (CODER, PLAN, DOC, RETRO, REVIEW) has at least one input mismatch with its coordinator agent's actual dispatch template. The contracts describe an idealized interface; the agents implement a different one. Skills built against the contract will receive unexpected inputs.

### Theme 3: WP Frontmatter Is Informally Specified

No single document defines the complete WP frontmatter schema. Fields are invented by agents (`review_status`, `docs_completed`, `docs_scope`, `review_cycles`) and documented piecemeal or not at all.l `base-handoff.schema.yaml` captures only 2 of 8+ fields. The Planner's WP template omits YAML frontmatter entirely (S-05).

---

## Summary

| Severity | Count | IDs |
|----------|-------|-----|
| Critical | 5 | C-01 through C-05 |
| Significant | 12 | S-01 through S-12 |
| Moderate | 16 | M-01 through M-16 |
| Minor | 10 | N-01 through N-10 |
| **Total** | **43** | |

### Priority Fix Order

1. **C-01** — Fix Orchestrator Decision Table row citations (wrong routing paths)
2. **C-02, C-03** — Add `brief_path` to Ideation and Brainstorming handoff prompts (pipeline bootstrap broken)
3. **C-04** — Resolve `code-implementation` dispatch architecture (WP-batch vs single-task)
4. **C-05** — Align `review_cycles` threshold across Orchestrator and reference doc
5. **S-01** — Add `lane: doing` branch to Coder Step 0 (resume broken)
6. **S-05, S-06** — Define WP frontmatter template in Planner
7. **S-07** — Either wire orphaned schemas into target agents or remove them
8. **S-02** — Complete debug dispatch inputs
9. **S-08, S-09** — Fix Review Coordinator spec promotion and commit policy contradictions
10. **S-03, S-04** — Fix Planner directory and schema issues
11. **S-10, S-11** — Add Docs Agent schema validation and fix input count
12. **S-12** — Guard `lane: to_do` + no FB-XX edge case
13. Moderate items by descending impact
14. Minor items at discretion
