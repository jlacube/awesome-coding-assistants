---
name: code-debug
description: "Test failure diagnosis, source code fixes, and regression detection"
argument-hint: "Invoked by Coder Coordinator - do not call directly"
---

# code-debug

> **Phase**: 5 (Conditional Debug)
> **Common contract**: `.github/skills/CODER-SKILL-CONTRACT.md`
> **Spec refs**: FR-034, FR-035, FR-036, FR-037

This skill is dispatched by the Coder Coordinator during Phase 5, only when unit or integration tests fail. It reads failing test output, diagnoses root causes, fixes source code (preferring source fixes over test fixes), re-runs all tests to verify fixes and detect regressions, and reports results. The coordinator controls the retry loop (max 3 attempts) -- this skill runs once per invocation.

---

## Input Contract (FR-017)

### Standard Inputs

| # | Input | Description |
|---|-------|-------------|
| 1 | `skill_path` | Path to this SKILL.md file |
| 2 | `wp_path` | Path to the WP file being implemented |
| 3 | `contracts_dir` | Path to contract files for this WP (`.sdd/plans/contracts/<WP-slug>/`) |
| 4 | `spec_path` | Path to the source spec file |
| 5 | `patterns` | Active code-domain patterns to avoid (from `code-patterns.md`) |
| 6 | `target_language` | Programming language (e.g., TypeScript, Python) |
| 7 | `target_framework` | Framework (e.g., Express, FastAPI, React) |
| 8 | `task_list` | Tasks with acceptance criteria and spec refs |

### Debug-Specific Inputs

| # | Input | Description |
|---|-------|-------------|
| 9 | `test_output` | Failing test output: test names, error messages, stack traces from the most recent test run |
| 10 | `source_file_list` | List of source files relevant to the failing tests |
| 11 | `debug_attempt` | Current debug attempt number (1, 2, or 3). Indicates urgency and remaining budget. |

The `test_output` input is the raw output from the test runner (unit + integration). It includes the names of failing tests, assertion error messages, and stack traces. The coordinator captures this from the prior test skill invocations.

The `debug_attempt` counter tells this skill how many attempts remain. At attempt 3, the skill should be more thorough in its diagnosis and consider escalating if the root cause is unclear.

---

## Execution Sequence (FR-018)

1. **Read SKILL.md** -- Load this file for debugging instructions, safety constraints, and reporting format
2. **Read WP + contracts** -- Read the WP file and contract files to understand the intended behavior and interface contracts
3. **Read spec sections** -- Read the spec sections referenced by the failing tests' tasks for requirements context
4. **Execute diagnosis and fix** -- Diagnose root causes, apply fixes, re-run all tests (Steps 1-5 below)
5. **Report results** -- Report fixed tests, still-failing tests, regressions, and files modified back to the coordinator

---

## Output Contract (FR-019)

Report to the coordinator with these fields:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `status` | enum | `success` or `failure` | Skill outcome (`success` = all tests pass, `failure` = tests still failing or cannot diagnose) |
| `files_modified` | list(string) | file paths | Files created or changed during debugging |
| `tasks_completed` | list(string) | T<NN>-XX format | Tasks whose failing tests are now fixed |
| `test_results` | object | `pass_count`, `fail_count`, `coverage_pct` | Test run summary after fixes |
| `issues` | list(string) | free text | Problems encountered during debugging |
| `failure_reason` | string | nullable | Why the skill failed (if status is `failure`) |

### Debug-Specific Output Fields

| Field | Type | Description |
|-------|------|-------------|
| `tests_fixed` | list(string) | Test names that were previously failing and now pass |
| `tests_still_failing` | list(object) | Tests still failing: `{name, error, diagnosis}` for each |
| `regressions` | list(string) | Test names that were passing before fixes but now fail |

The coordinator uses these fields to decide:
- **All tests pass** (`fail_count == 0`, `regressions` empty): Done. Do not dispatch again.
- **Tests still failing** (`fail_count > 0`, `debug_attempt < 3`): Retry with incremented attempt counter.
- **Cannot diagnose** (`status == failure`, `failure_reason` set): Escalate to human immediately.
- **3 attempts exhausted** (`debug_attempt == 3`, `fail_count > 0`): Coordinator escalates to human.
