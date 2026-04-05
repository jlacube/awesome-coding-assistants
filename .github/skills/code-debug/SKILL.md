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

---

## Step 1 -- Read and Categorize Failing Tests (FR-034.1)

Parse the `test_output` input to extract every failing test. For each failure, record:

| Field | Source | Example |
|-------|--------|---------|
| Test name | Test runner output | `test_create_user_returns_hashed_password` |
| Error type | Exception/assertion class | `AssertionError`, `TypeError`, `AttributeError` |
| Error message | Assertion diff or exception message | `expected 'string' but got 'number'` |
| Stack trace | Full traceback | File, line number, function name |
| Test file | Stack trace or test runner | `tests/unit/test_user.py:42` |

### 1a. Group Failures by Root Cause

Multiple test failures often share a single root cause. Group failures that:
- Reference the same source file and function
- Produce the same error type
- Fail on the same assertion pattern (e.g., all get a `TypeError` from the same function)

Fixing the shared root cause should resolve all tests in the group. Prioritize groups with the most failures first.

### 1b. Identify the Relevant Source Code (FR-034.2)

For each failure group, locate the source code under test:

1. Read the stack trace to find the source file and line number where the error originates
2. Read the test file to identify which function/class/module is being tested
3. Read the source file containing the function under test
4. Read enough surrounding context (the full function, class, or module) to understand the logic

### 1c. Read Contract and Spec Context (FR-034.3)

For each failure group, load the relevant contracts and spec sections:

1. Read contract files from `contracts_dir` that define the interface, data schema, or error catalog for the failing code
2. Read the spec sections referenced by the task that owns the failing tests
3. Identify the expected behavior according to the spec and contracts -- this is the source of truth for determining whether the source code or test code is wrong

---

## Step 2 -- Diagnose Root Causes (FR-034.4)

For each failure group, determine the root cause by answering these questions in order:

### 2a. Diagnosis Decision Tree

```
1. Does the source code match the contract file signatures exactly?
   - Function names, parameter names, types, return types
   - Data entity field names, types, defaults, validation rules
   - Error codes, messages, HTTP status codes
   If NO --> Root cause: contract deviation in source code

2. Does the source code implement the spec's SHALL obligations?
   - All preconditions checked
   - All postconditions produced
   - All error paths handled
   If NO --> Root cause: missing or incorrect spec implementation

3. Does the test correctly reflect the spec's expected behavior?
   - Test assertions match spec acceptance scenarios
   - Test inputs match spec preconditions
   - Expected outputs match spec postconditions
   If NO --> Root cause: incorrect test (test does not match spec)

4. Is there a logic error in the source code?
   - Off-by-one errors, wrong operator, missing null check
   - Incorrect control flow (wrong branch, missing case)
   - Data transformation error (wrong field, wrong format)
   If YES --> Root cause: implementation bug
```

### 2b. Document Each Diagnosis

For each failure group, record:

- **Root cause**: One-sentence description of what is wrong
- **Evidence**: The specific code, contract, or spec text that proves the diagnosis
- **Category**: `contract-deviation`, `missing-implementation`, `incorrect-test`, `logic-error`, or `unknown`
- **Fix location**: Whether the source code or test code needs to change
- **Spec justification**: The FR or acceptance scenario that defines the correct behavior
