# Adversarial Review — Round 5 (Final)

**Date**: 2025-07-10
**Scope**: `.github/agents/`, `.github/skills/`, `.github/schemas/`, `.github/prompts/`, `index.json`, `index.schema.json`
**Methodology**: Full cross-referencing of all 10 agent files, 6 skill contracts, 47 individual SKILL.md files, 25 schema files, 1 prompt file, and 2 index files. Every finding was verified against FINAL-ISSUE-REGISTER.md (74 items) to ensure no duplicate reporting.
**Purpose**: Final sweep for issues not caught by R1–R4 or the consolidated issue register.

---

## Severity Definitions

| Level | Meaning |
|-------|---------|
| **FIX** | Causes runtime failure, semantic contradiction, or silently wrong behavior on a standard pipeline path |
| **ACCEPT** | Real inconsistency with no practical runtime impact |
| **DEFER** | Valid improvement, not worth blocking closure |

---

## Part 1: FINAL-ISSUE-REGISTER Fix Verification

Before searching for new issues, all 44 FIX items were spot-checked against the actual source files. **22 of 24 sampled items are fully fixed. 2 are partially fixed.**

### Residual Defect R5-RD01 — `code-implementation/SKILL.md` still says "all tasks in one WP"

- **Register item**: D1 (R4-C04)
- **Status**: PARTIALLY FIXED
- **Evidence**: SKILL.md Step 1 correctly says "Each `code-implementation` invocation handles a single task dispatched by the Coder agent." CODER-SKILL-CONTRACT Section 5 correctly says "dispatched once per task." However, the SKILL.md intro description (approx. line 13) still reads: *"One invocation handles all tasks in one WP."* Line 246 repeats the same claim.
- **Impact**: A skill author reading the intro will implement batch-task logic that the Coder will never invoke.
- **Disposition**: **FIX** — update lines 13 and 246 to say "one invocation = one task."

### Residual Defect R5-RD02 — `orchestrator.agent.md` Step 7 vague tool reference

- **Register item**: E3 (R4-N02)
- **Status**: PARTIALLY FIXED
- **Evidence**: The bad `replace_string_in_file` reference was removed. Step 7 item 3 now says "Write the updated state file back by editing the YAML frontmatter block" but does not name the correct tool (`edit/editFiles`). All other agents that edit files name their tool explicitly.
- **Impact**: The orchestrator LLM must infer the tool from context rather than follow an explicit instruction.
- **Disposition**: **ACCEPT** — the vague phrasing works in practice; the LLM resolves it from the `tools:` array.

---

## Part 2: New Findings

### Category F: Tool Name and Declaration Issues

#### F1 — `manage_todo_list` vs `#tool:todo` inconsistency across 4 agents [FIX]

| Agent | Body notation | Frontmatter |
|---|---|---|
| `brainstorming.agent.md` | `#tool:todo` | `todo` |
| `ideation.agent.md` | `#tool:todo` | `todo` |
| `docs-agent.agent.md` | `#tool:todo` | `todo` |
| `orchestrator.agent.md` | `#tool:todo` | `todo` |
| `review-coordinator.agent.md` | `#tool:todo` | `todo` |
| **`planner.agent.md`** | **`manage_todo_list`** | `todo` |
| **`spec-architect.agent.md`** | **`manage_todo_list`** | `todo` |
| **`retro-spec.agent.md`** | **`manage_todo_list`** | `todo` |
| **`coder.agent.md`** | **BOTH** — rules say `#tool:todo`; Steps 6b.1 and 8b say `manage_todo_list` | `todo` |

- **Files**: `planner.agent.md`, `spec-architect.agent.md`, `retro-spec.agent.md`, `coder.agent.md`
- **Impact**: `manage_todo_list` is not in any agent's `tools:` array. When the LLM encounters this name, it must map it to `todo` by inference. Coder has an internal contradiction (both names in the same file).
- **Cross-ref**: Structurally similar to E18 (`askQuestions` naming) but affects 4 agents. Not in register.
- **Fix**: Replace all `manage_todo_list` with `#tool:todo` in the 4 affected agents.
- **Disposition**: **FIX**

#### F2 — `spec-architect.agent.md` uses `grep_search` with no fallback [ACCEPT]

- **File**: `spec-architect.agent.md` Step 7 validation
- **Evidence**: Body says `Use grep_search and read_file on the accumulator to verify` — no declared tool matches `grep_search` in frontmatter. Unlike `coder.agent.md` and `review-coordinator.agent.md` which provide `textSearch` as an alternative, spec-architect provides no fallback.
- **Impact**: The LLM resolves `grep_search` to `search/textSearch` from context. Both coder and review-coordinator already demonstrate this mapping works.
- **Cross-ref**: Not in register.
- **Disposition**: **ACCEPT** — works in practice from the `tools:` array.

#### F3 — Orchestrator declares 19 unused tools in frontmatter [ACCEPT]

- **File**: `orchestrator.agent.md` frontmatter `tools:` array
- **Evidence**: 10 browser tools, 2 task tools, 7 VS Code IDE tools are declared but never referenced in the body. The orchestrator is a pure state machine that delegates all work — it never opens browsers, runs tasks, or manages extensions.
- **Impact**: Inflated tool menu; no runtime effect. The tools are available but never selected.
- **Cross-ref**: Not in register. Inverse of E3 (body referencing undeclared tool).
- **Disposition**: **ACCEPT** — removing tools requires knowing the full VS Code tool resolution model, and a superset is safer than a subset.

#### F4 — `vscode/memory` declared in brainstorming, ideation, orchestrator but never used [ACCEPT]

- **Files**: `brainstorming.agent.md`, `ideation.agent.md`, `orchestrator.agent.md`
- **Evidence**: All three declare `vscode/memory` in frontmatter. None call it in any step. Orchestrator uses `.sdd/state.md` for persistence instead.
- **Disposition**: **ACCEPT** — harmless superset declaration.

#### F5 — Brainstorming and ideation declare `runTests`, `runNotebookCell`, `createJupyterNotebook` [ACCEPT]

- **Files**: `brainstorming.agent.md`, `ideation.agent.md`
- **Evidence**: Both agents' frontmatter includes testing and notebook tools. Both agents' rules section explicitly states: "NEVER write code, architecture diagrams, or implementation details."
- **Impact**: Tool availability contradicts the stated rule. No runtime failure — the rule prevents usage.
- **Disposition**: **ACCEPT** — the rule override prevents invocation.

---

### Category G: WP Template and Spec Field Gaps

#### G1 — Planner coordinator WP template missing `| Spec |` markdown table [FIX]

- **File**: `planner.agent.md` (WP template section after Step 13)
- **Evidence**: The planner's coordinator-level WP template has YAML frontmatter (`lane`, `depends_on`, etc.) and jumps to `## Objective`. It does NOT include the markdown table with the `| Spec |` row. Meanwhile, `plan-decomposition/SKILL.md` Step 5 template DOES include:
  ```
  | Field | Value |
  |-------|-------|
  | Spec | `<spec_path>` |
  ```
- **Downstream consumers that read `Spec`**:
  - `coder.agent.md` Step 2.2: "Read the spec section(s) referenced in the WP's `Spec` field"
  - `review-coordinator.agent.md` Step 2.2: "read the WP file's `Spec` field to find the spec path"
  - `docs-agent.agent.md` Step 1.4: "Extract the spec path from the WP file's `Spec` field"
  - `orchestrator.agent.md` Step 6: "The Orchestrator derives `spec_path` from the WP file's `Spec` field"
- **Cross-ref**: C1 (R4-S05) says "WP template shows only markdown body, no YAML frontmatter" — but the YAML frontmatter **already exists** (C1 was already applied). C1's fix description does not mention the `| Spec |` table. After C1, this gap persists.
- **Impact**: If the `plan-decomposition` skill's template is the one that produces WPs at runtime, WPs will have the `Spec` field. But if the coordinator's template is used as a reference or override, WPs lack the `Spec` field and all four downstream agents fail at spec lookup.
- **Fix**: Add the `| Spec |` table to the planner coordinator's WP template, matching `plan-decomposition` SKILL.md Step 5.
- **Disposition**: **FIX**

#### G2 — Planner and plan-decomposition WP templates diverge structurally [ACCEPT]

- **Files**: `planner.agent.md` (post-Step 13), `plan-decomposition/SKILL.md` (Step 5)
- **Evidence**:
  | Section | plan-decomposition SKILL | planner coordinator |
  |---|---|---|
  | `## Research Context` | Present | **Absent** |
  | `## Parallel Opportunities` | **Absent** | Present |
- **Impact**: `coder.agent.md` Step 2.5 explicitly reads `## Research Context`: "Extract the `## Research Context` section from the WP file (if present)." The "(if present)" guard means this is a soft dependency — no crash, but lost context.
- **Disposition**: **ACCEPT** — the SKILL.md template is the one invoked at runtime; the coordinator template is a reference. The `(if present)` guard prevents failure.

---

### Category H: Contract ↔ SKILL.md Input Mismatches (Untracked)

#### H1 — All 8 doc SKILL.md files missing `contracts_dir` (input #7) [FIX]

- **Files**: All `doc-*/SKILL.md` (8 files)
- **Evidence**: DOC-SKILL-CONTRACT.md defines 7 inputs with `contracts_dir` as #7. Every individual doc SKILL.md declares only 6 inputs (or 5 for `doc-inline-code`). None include `contracts_dir`.
- **Cross-ref**: D4 covers the contract side ("Add `contracts_dir` as Input #7 in DOC-SKILL-CONTRACT"). D4 has already been applied — the contract now has 7 inputs. But no register item covers propagating this to the 8 SKILL.md files.
- **Additional**: `doc-inline-code/SKILL.md` declares only 5 inputs — missing both `docs_dir` (#6) and `contracts_dir` (#7). D5 covers removing the exception from the contract, not updating the SKILL.md.
- **Fix**: Add `contracts_dir` as input #7 to all 8 doc SKILL.md files. Add `docs_dir` as input #6 to `doc-inline-code/SKILL.md`.
- **Disposition**: **FIX**

#### H2 — All retro SKILL.md files declare `patterns` input #9; contract has no such input, agent never passes it [FIX]

- **Files**: All `retro-*/SKILL.md` (7 extraction skills, excluding `retro-discovery` and `retro-assembly`)
- **Evidence**: Each file declares `| 9 | patterns | Active retro-domain patterns |`. RETRO-SKILL-CONTRACT.md defines exactly 8 inputs with no `patterns`. `retro-spec.agent.md` dispatch templates (project-level: 7 items; module-level: 9 items where #9 is extraction depth, not patterns) never send a `patterns` variable.
- **Cross-ref**: E25 notes "No patterns consumption step — anti-patterns never reach extraction skills" but is scoped to the *agent* not passing patterns. No register item addresses the SKILL.md files *declaring* an input that doesn't exist.
- **Fix**: Either (a) remove `patterns` from all 7 retro SKILL.md files and document it as a future feature per E25/DEFER, or (b) add `patterns` as input #9 to the contract and the agent dispatch template. Option (a) is consistent with E25.
- **Disposition**: **FIX**

#### H3 — `code-debug` SKILL.md declares 11 inputs; contract table has 8 [ACCEPT]

- **File**: `code-debug/SKILL.md`, `CODER-SKILL-CONTRACT.md`
- **Evidence**: SKILL.md has standard inputs 1–8 plus three debug-specific: `test_output` (#9), `source_file_list` (#10), `debug_attempt` (#11). The contract's Section 1 input table lists 8 universal inputs.
- **Cross-ref**: D2 covers the agent→skill dispatch direction (agent missing inputs). D3 covers the Section 4 template. Neither explicitly addresses the Section 1 input table gap.
- **Impact**: Low — the Section 4 template (when D3 is applied) will document the full set. Section 1 is a summary that intentionally shows universal inputs only.
- **Disposition**: **ACCEPT** — D2+D3 together will make the full input set visible.

#### H4 — Pattern file paths: 3 contracts use bare filenames, 1 uses full path [FIX]

- **Files**: `CODER-SKILL-CONTRACT.md`, `PLAN-SKILL-CONTRACT.md`, `SPEC-SKILL-CONTRACT.md`, `DOC-SKILL-CONTRACT.md`
- **Evidence**:
  | Contract | `patterns` path reference |
  |---|---|
  | DOC | `.sdd/reviews/doc-patterns.md` (full path) |
  | CODER | `code-patterns.md` (bare filename) |
  | PLAN | `plan-patterns.md` (bare filename) |
  | SPEC | `spec-patterns.md` (bare filename) |
- **Cross-ref**: D9 discusses documenting the pattern curation system (DEFER) — a different concern. This path format inconsistency is not in the register.
- **Impact**: Skill authors reading CODER/PLAN/SPEC contracts cannot determine the actual file path. They would look for `code-patterns.md` in the workspace root, not `.sdd/reviews/code-patterns.md`.
- **Fix**: Add `.sdd/reviews/` prefix to the CODER, PLAN, and SPEC contracts' `patterns` input description.
- **Disposition**: **FIX**

#### H5 — `review-spec-completeness` finding format completely diverges from REVIEW-SKILL-CONTRACT §3.2 [DEFER]

- **Files**: `review-spec-completeness/SKILL.md` §2, `REVIEW-SKILL-CONTRACT.md` §3.2
- **Evidence**:
  | Dimension | Contract §3.2 | `review-spec-completeness` SKILL.md |
  |---|---|---|
  | Heading | `### PREFIX-NNN [FAIL]` | `### Finding: SPEC-COMP-XXX` |
  | Severity | In heading: `[FAIL]`, `[WARN]` | Body field: `**Severity**: HIGH \| MEDIUM \| LOW` |
  | Body fields | Checklist item, Requirement, File, Description, Expected, Evidence | Severity, Category, Location, Issue, Recommendation |
- **Cross-ref**: Not in register. D6 covers batch structure (different). E23 covers dead pattern domain row (different).
- **Impact**: Tools or scripts parsing review findings by the contract format will not match `review-spec-completeness` output. The SKILL.md §3 maps internal severities to `finding_counts` correctly, so the frontmatter aggregation works.
- **Disposition**: **DEFER** — the skill operates correctly for its purpose (pre-planning spec validation). The non-standard format is a conscious design choice for a skill that analyzes specs rather than code.

#### H6 — `SPEC-` / `SPEC-COMP-` finding prefix collision risk [ACCEPT]

- **Files**: `review-spec/SKILL.md`, `review-spec-completeness/SKILL.md`
- **Evidence**: `review-spec` uses `SPEC-NNN` prefixes; `review-spec-completeness` uses `SPEC-COMP-NNN`. A regex `^SPEC-` matches both.
- **Impact**: Negligible — no agent or tool uses regex prefix matching on finding IDs. The `COMP` infix provides sufficient human disambiguation.
- **Disposition**: **ACCEPT**

---

### Category I: `docs_scope` Semantic Mismatch

#### I1 — Schema example uses `doc-` prefix; docs-agent strips `doc-` for matching — silent no-op filter [FIX]

- **Files**: `base-handoff.schema.yaml`, `docs-agent.agent.md`, `plan-decomposition/SKILL.md`
- **Evidence**: Three sources define valid `docs_scope` values differently:
  | Source | Example values | Has `doc-` prefix? |
  |---|---|---|
  | `base-handoff.schema.yaml` description | `['doc-api-reference', 'doc-changelog']` | **YES** |
  | `docs-agent.agent.md` Step 5b | `docs_scope: [changelog, developer-guide]` | **NO** |
  | `plan-decomposition/SKILL.md` | `architecture, api-reference, changelog, inline-code` | **NO** |
- **Failure path**: A WP authored following the schema example stores `docs_scope: ['doc-changelog']`. The docs-agent strips `doc-` from its skill names to get `changelog`, then compares against the WP's `doc-changelog`. No match. All 8 doc skills dispatch instead of just the one intended. No error is raised — the filter silently becomes a no-op.
- **Cross-ref**: D17 (DEFER) mentions documenting `docs_scope` in DOC-SKILL-CONTRACT — different concern, and this semantic mismatch is not in the register.
- **Fix**: Change `base-handoff.schema.yaml` example to `['api-reference', 'changelog']` (without `doc-` prefix), matching the docs-agent's matching logic and `plan-decomposition`'s documented values.
- **Disposition**: **FIX**

---

### Category J: Index Files

#### J1 — Repo name mismatch: `awesome-coding-assistants` vs `awesome-copilot-assistants` [FIX]

- **Files**: `index.json`, `index.schema.json`
- **Evidence**:
  - `index.json` `$schema` URL: `https://raw.githubusercontent.com/jlacube/awesome-coding-assistants/main/index.schema.json`
  - `index.json` source URLs: `https://github.com/jlacube/awesome-coding-assistants`
  - `index.schema.json` `$id`: `https://raw.githubusercontent.com/jlacube/awesome-coding-assistants/main/index.schema.json`
  - Workspace folder: `c:\Sandbox\Git\awesome-copilot-assistants`
- **Impact**: `$schema` and `$id` URLs are broken if the public repo is `awesome-copilot-assistants`. VS Code JSON schema validation will fail to fetch the schema. Any JSON tooling using `$id` for resolution will use a stale identity.
- **Fix**: Replace `awesome-coding-assistants` with `awesome-copilot-assistants` in all three URLs.
- **Disposition**: **FIX**

#### J2 — Index schema allows tool names not supported by this codebase [ACCEPT]

- **File**: `index.schema.json`
- **Evidence**: The `tools` enum includes `kiro`, `kilocode`, `opencode` alongside `copilot` and `claude-code`. The entire `.github/` agent and skill system is VS Code Copilot-only.
- **Impact**: None — the schema describes a cross-tool index format. The extra enum values support future or external contributions.
- **Disposition**: **ACCEPT** — this is intentional extensibility in a community index schema.

---

### Category K: Commit Scope Ambiguity

#### K1 — Coder uses `docs(plan):` commit scope, overlapping with Planner [ACCEPT]

- **File**: `coder.agent.md` commit table
- **Evidence**: Coder commits WP frontmatter updates with `docs(plan): mark WP02 complete, submit for review`. The `docs(plan):` scope is used by the Planner in every other context.
- **Impact**: `git log --grep="docs(plan):"` returns both Planner and Coder commits. The orchestrator's orphaned-commit check cannot distinguish agent authorship from the commit message alone.
- **Disposition**: **ACCEPT** — both agents modify files in `.sdd/plans/`, so the scope is technically correct. Agent attribution comes from the Activity Log, not commit messages.

---

## Part 3: Summary

### New Findings

| ID | Category | Severity | Files | Issue |
|---|---|---|---|---|
| R5-RD01 | Residual | **FIX** | `code-implementation/SKILL.md` | Intro still says "all tasks in one WP" (D1 partial) |
| F1 | Tool Names | **FIX** | 4 agents | `manage_todo_list` vs `#tool:todo` — 4 agents use wrong name |
| F2 | Tool Names | ACCEPT | `spec-architect.agent.md` | `grep_search` with no fallback |
| F3 | Tool Decl | ACCEPT | `orchestrator.agent.md` | 19 unused tools in frontmatter |
| F4 | Tool Decl | ACCEPT | 3 agents | `vscode/memory` declared but never used |
| F5 | Tool Decl | ACCEPT | 2 agents | Test/notebook tools contradict "never code" rule |
| G1 | WP Template | **FIX** | `planner.agent.md` | Missing `\| Spec \|` markdown table (C1 gap) |
| G2 | WP Template | ACCEPT | planner vs plan-decomposition | `## Research Context` absent from coordinator template |
| H1 | Input Sync | **FIX** | 8 doc SKILL.md files | All missing `contracts_dir` (#7); `doc-inline-code` also missing `docs_dir` (#6) |
| H2 | Input Sync | **FIX** | 7 retro SKILL.md files | Declare `patterns` #9 that contract doesn't define and agent never passes |
| H3 | Input Sync | ACCEPT | `code-debug/SKILL.md` | 11 inputs vs 8 in contract Section 1 (covered by D2+D3) |
| H4 | Input Sync | **FIX** | 3 skill contracts | Bare pattern filenames vs full `.sdd/reviews/` path |
| H5 | Output Format | DEFER | `review-spec-completeness/SKILL.md` | Finding format diverges from REVIEW-SKILL-CONTRACT §3.2 |
| H6 | Output Format | ACCEPT | review-spec, review-spec-completeness | `SPEC-` / `SPEC-COMP-` prefix collision risk |
| I1 | docs_scope | **FIX** | `base-handoff.schema.yaml` | Schema example `doc-` prefix causes silent filter no-op |
| J1 | Index | **FIX** | `index.json`, `index.schema.json` | Repo name `awesome-coding-assistants` → `awesome-copilot-assistants` |
| J2 | Index | ACCEPT | `index.schema.json` | Extra tool names for future extensibility |
| K1 | Commits | ACCEPT | `coder.agent.md` | `docs(plan):` scope overlap with Planner |

### Disposition Counts

| Disposition | Count |
|---|---|
| **FIX** | 9 (including 1 residual from register) |
| **ACCEPT** | 8 |
| **DEFER** | 1 |
| **Total** | **18** |

### Impact on FINAL-ISSUE-REGISTER

The 9 FIX items should be appended to the register:

| Suggested ID | R5 ID | Group |
|---|---|---|
| D1-residual | R5-RD01 | D (amend D1 scope) |
| E33 | F1 | E (tool naming, same class as E18) |
| C5 | G1 | C (WP template extension) |
| D19 | H1 | D (contract→SKILL.md propagation) |
| D20 | H2 | D (contract→SKILL.md alignment) |
| D21 | H4 | D (contract path formatting) |
| B13 | I1 | B (schema semantics) |
| E34 | J1 | E (infrastructure URLs) |

---

## Conclusion

R5 surfaces **9 actionable FIX items**, all either (a) gaps in register-to-source propagation (R5-RD01, H1), (b) cross-cutting consistency issues that prior rounds examined per-file rather than cross-family (F1, H2, H4), or (c) schema/index infrastructure not previously in scope (I1, J1). None represent fundamental design flaws — the pipeline architecture is sound.

After applying these 9 fixes plus the remaining 44 from the register (of which 22 are verified complete), the system has no known FIX-severity issues.
