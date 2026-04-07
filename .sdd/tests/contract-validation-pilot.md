# Contract Validation Pilot

## Overview

This document describes integration test scenarios that validate the Coder agent's contract-first workflow against a small, real codebase (not the pipeline's own markdown/YAML files). The pipeline has been used to build itself -- these scenarios confirm that it also works for real source code projects.

Three Coder capabilities are exercised:

1. **Contract-first implementation** -- reading contract files and implementing to match
2. **Coverage threshold enforcement** -- verifying tests meet configured thresholds
3. **Debug retry loops** -- handling test failures and retrying with diagnosis

## Prerequisites

- The SDD pipeline is functional and has been used to build itself
- A small real codebase exists with at least one module (TypeScript or Python)
- Contract files (interfaces, data-schemas) exist for the target module
- The Coder agent mode is available and configured

## Test Scenarios

### Scenario 1: Contract-First Implementation

**Capability**: The Coder reads contract files and produces implementation that matches them exactly.

**Setup**:
1. Create a minimal TypeScript project with a single module (e.g., `src/calculator.ts`)
2. Define a contract file `interfaces.ts` in `.sdd/plans/contracts/<WP-slug>/`:
   ```typescript
   export interface CalculatorInput {
     a: number;
     b: number;
     operation: 'add' | 'subtract' | 'multiply' | 'divide';
   }

   export interface CalculatorResult {
     value: number;
     operation: string;
   }

   export function calculate(input: CalculatorInput): CalculatorResult;
   ```
3. Define a WP with a task referencing this contract file
4. Run the Coder agent against the WP

**Expected Outcome**:
- The Coder creates `src/calculator.ts` implementing the `calculate` function
- The function signature matches the contract exactly: same parameter name (`input`), same type (`CalculatorInput`), same return type (`CalculatorResult`)
- Field names in `CalculatorInput` and `CalculatorResult` match the contract verbatim
- The implementation handles all four operations including the `divide` edge case (division by zero)

**Pass/Fail Criteria**:
- **PASS**: Implementation file exists, function signature matches contract exactly, all interface fields match, basic logic is correct
- **FAIL**: Function signature differs from contract (renamed parameters, changed types), missing fields, or implementation file not created

**Verification Method**: Diff the implemented function signature against the contract file. Run `tsc --noEmit` to verify type compatibility. Manually inspect the implementation for correctness.

---

### Scenario 2: Coverage Threshold Enforcement

**Capability**: The Coder's test skills enforce configured coverage thresholds against real code.

**Setup**:
1. Use the same TypeScript project from Scenario 1 with a working implementation
2. Configure the WP with coverage thresholds: `coverage_code: 60`, `coverage_branch: 70`
3. Run the Coder agent, which dispatches `code-unit-tests` to write tests
4. Verify the test skill checks coverage after test execution

**Expected Outcome**:
- The Coder dispatches the `code-unit-tests` skill
- Tests are generated for `src/calculator.ts`
- After test execution, coverage is measured
- If coverage is below thresholds (60% code, 70% branch), the Coder re-dispatches the test skill to add more tests
- Final coverage meets or exceeds thresholds

**Pass/Fail Criteria**:
- **PASS**: Coverage report is generated, thresholds are checked, and if below threshold the Coder adds more tests until thresholds are met
- **FAIL**: No coverage check occurs, thresholds are ignored, or the Coder proceeds despite below-threshold coverage

**Verification Method**: Read the coverage report output. Verify the Coder's activity log shows threshold comparison. If re-dispatch occurred, verify the second test run improved coverage.

**Note**: This scenario connects to WP44 (configurable coverage thresholds). The default thresholds are 80% code / 90% branch per the Coder agent's Step 9 (FR-014.1). The scenario uses lower thresholds (60/70) to test that custom values are respected.

---

### Scenario 3: Debug Retry Loop

**Capability**: The Coder's debug skill diagnoses test failures and retries with fixes.

**Setup**:
1. Use the same TypeScript project with a working implementation and tests
2. Introduce a deliberate bug: change the `add` case to return `a - b` instead of `a + b`
3. Run the Coder agent against a WP that includes both implementation and test tasks
4. Observe the debug retry behavior

**Expected Outcome**:
- Unit tests fail due to the deliberate bug
- The Coder detects test failures and dispatches the `code-debug` skill
- The debug skill diagnoses the root cause (incorrect operation in `add` case)
- The debug skill fixes the source code (not the test)
- All tests pass after the fix
- If the first debug attempt fails, the Coder retries up to the maximum retry count (3 attempts per the Coder's Step 7)

**Pass/Fail Criteria**:
- **PASS**: Debug skill is dispatched on test failure, diagnoses the correct root cause, fixes the source code, and tests pass within the retry limit
- **FAIL**: Debug skill is not dispatched, fixes the test instead of the source, deletes failing tests, or exceeds the retry limit without resolution

**Verification Method**: Read the Coder's activity log for debug dispatch entries. Verify the fix targets source code (not test files). Run tests to confirm they pass. Check that retry count does not exceed 3.

## Results

### Scenario 1: Contract-First Implementation

**Status**: Not yet executed

**Reason**: No suitable real codebase exists in the current workspace. The workspace contains only the SDD pipeline itself (markdown/YAML files), which is not a valid target for contract-first implementation testing per FR-055.

### Scenario 2: Coverage Threshold Enforcement

**Status**: Not yet executed

**Reason**: No suitable real codebase exists in the current workspace. Coverage enforcement requires executable source code with a test runner and coverage tooling (e.g., Jest with Istanbul for TypeScript). The current workspace has no such project.

### Scenario 3: Debug Retry Loop

**Status**: Not yet executed

**Reason**: No suitable real codebase exists in the current workspace. Debug retry testing requires a project with executable tests where a deliberate bug can be introduced and diagnosed. The current workspace has no such project.

## Findings

No findings to report at this time -- all scenarios are in "Not yet executed" status.

### Finding Categories

When scenarios are executed, each discovered issue will be categorized as:

| Category | Definition | Example |
|----------|------------|---------|
| **Blocking** | Prevents real-world use of the capability | Coder ignores contract files entirely |
| **Degraded** | Capability works but suboptimally | Coder matches signatures but misses edge cases from the contract |
| **Cosmetic** | Minor issue that does not affect functionality | Generated code formatting differs from codebase conventions |

### Known Limitations

- **No suitable test codebase**: The current workspace contains only the SDD pipeline (markdown and YAML files). Contract-first implementation, coverage enforcement, and debug retry testing require a real source code project (TypeScript, Python, or similar). This is recorded per FR-057.
- **Proposed future validation**: Execute these scenarios against a small sample project when one is available. A minimal TypeScript or Python project with 1-2 modules would be sufficient to validate all three capabilities.
