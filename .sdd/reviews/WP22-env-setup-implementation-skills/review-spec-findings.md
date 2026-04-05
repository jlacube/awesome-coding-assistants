---
skill: review-spec
wp: WP22-env-setup-implementation-skills
date: 2026-04-05T15:00:00Z
status: WARN
files_reviewed:
  - .github/skills/code-env-setup/SKILL.md
  - .github/skills/code-implementation/SKILL.md
  - .github/skills/CODER-SKILL-CONTRACT.md
finding_counts:
  pass: 14
  warn: 1
  fail: 0
  na: 0
---

# review-spec Findings -- WP22

## In-Scope FRs

FR-017, FR-018, FR-019, FR-020 (020.1-020.6), FR-021, FR-022, FR-023, FR-024 (024.1-024.5), FR-025 (025.1-025.4), FR-026

---

### SPEC-001 [PASS]

**FR**: FR-017 (Input Contract)
**Classification**: Compliant
**Evidence**: Both `code-env-setup/SKILL.md` and `code-implementation/SKILL.md` include an Input Contract table listing all 8 required inputs: `skill_path`, `wp_path`, `contracts_dir`, `spec_path`, `patterns`, `target_language`, `target_framework`, `task_list`. Fields match `CODER-SKILL-CONTRACT.md` exactly.

---

### SPEC-002 [PASS]

**FR**: FR-018 (Execution Sequence)
**Classification**: Compliant
**Evidence**: Both skills define a 5-step execution sequence matching the spec: (1) Read SKILL.md, (2) Read WP + contracts, (3) Read spec sections, (4) Execute work, (5) Report results.

---

### SPEC-003 [PASS]

**FR**: FR-019 (Output Contract)
**Classification**: Compliant
**Evidence**: Both skills define Output Contract tables with all required fields: `status` (enum: success/failure), `files_modified` (list of paths), `tasks_completed` (list of task IDs), `test_results` (object with pass_count, fail_count, coverage_pct), `issues` (list), `failure_reason` (nullable string).

---

### SPEC-004 [PASS]

**FR**: FR-020.1 (Check existing environment)
**Classification**: Compliant
**Evidence**: `code-env-setup/SKILL.md` Step 1 covers Python (.venv/, venv/, pyproject.toml, requirements.txt, setup.py, Pipfile, conda), Node.js (node_modules/, package.json, lockfiles), Go (go.mod, go.sum), Rust (Cargo.toml, Cargo.lock), and generic (Makefile, Docker, .tool-versions).

---

### SPEC-005 [PASS]

**FR**: FR-020.2 (Create virtual environment)
**Classification**: Compliant
**Evidence**: Step 2 covers Python (`python -m venv .venv`), Node.js (npm/yarn/pnpm install), Go (`go mod download`), Rust (`cargo fetch`), and generic fallback. Version checks included for all languages.

---

### SPEC-006 [PASS]

**FR**: FR-020.3 (Install project dependencies)
**Classification**: Compliant
**Evidence**: Step 3 covers Python (pip, poetry, pipenv), Node.js (handled in Step 2), Go (`go mod download` + `go build`), Rust (`cargo build`). Error handling specified with halt-on-failure behavior.

---

### SPEC-007 [PASS]

**FR**: FR-020.4 (Run existing tests)
**Classification**: Compliant
**Evidence**: Step 5a specifies running test suites per language (pytest, npm test, go test, cargo test). Decision logic: pass -> continue, fail -> report as pre-existing issue, no tests -> not a failure.

---

### SPEC-008 [PASS]

**FR**: FR-020.5 (Verify application launch)
**Classification**: Compliant
**Evidence**: Step 5b covers application launch verification if entry point exists, with skip+documentation for libraries/plugins with no entry point.

---

### SPEC-009 [PASS]

**FR**: FR-020.6 (Document environment state)
**Classification**: Compliant
**Evidence**: Step 5c specifies Activity Log format with language version, framework version, venv type, dependency count, baseline test results, and coverage tooling status.

---

### SPEC-010 [PASS]

**FR**: FR-021 (Environment failure handling)
**Classification**: Compliant
**Evidence**: Step 6 implements the 3-part failure protocol: (6a) document what failed with exact error message, (6b) report failure to coordinator via output contract, (6c) coordinator escalation. Common failure scenarios table covers: missing runtime, version mismatch, dependency conflict, network error, missing system library, insufficient permissions.

---

### SPEC-011 [WARN]

**FR**: FR-022 (Coverage tooling)
**Classification**: Partial
**Evidence**: Step 4 correctly installs coverage tooling per language and the textual description states "80% code coverage, 90% branch coverage" as exact minimums. However, the Python pytest-cov/coverage.py configuration example only sets `fail_under = 80` (overall threshold). Coverage.py does not natively support a separate branch-specific `fail_under`. The Node.js configurations (Jest, nyc) correctly specify `branches: 90` as a separate threshold. The Python config gap means a project could pass with 80% overall coverage but branch coverage below 90%.
**File**: `.github/skills/code-env-setup/SKILL.md` lines 213-230 (Python config examples)
**Expected**: Either note the coverage.py limitation and recommend a post-test verification step for branch coverage, or include a script/hook that checks branch coverage separately against the 90% threshold.

---

### SPEC-012 [PASS]

**FR**: FR-023 (Contract-first implementation)
**Classification**: Compliant
**Evidence**: `code-implementation/SKILL.md` Step 3 covers all 7 sub-requirements: (3a) read spec refs and acceptance criteria, (3b) copy interface/type definitions verbatim, (3c) implement API endpoints, (3d) implement state transitions, (3e) implement error handling, (3f) implement all error paths, (3g) follow codebase conventions, (3h) check off acceptance criteria.

---

### SPEC-013 [PASS]

**FR**: FR-024 (Contract matching)
**Classification**: Compliant
**Evidence**: Step 2 table lists all 5 contract types (interfaces, data-schemas, api-contracts, state-machines, error-catalog). Steps 3b-3e enforce verbatim matching for each type. Missing contract file handling halts execution. Example shows exact signature matching.

---

### SPEC-014 [PASS]

**FR**: FR-025 (Constraints)
**Classification**: Compliant
**Evidence**: Constraints section explicitly lists all 4 SHALL NOT rules: (FR-025.1) no over-engineering with specific examples, (FR-025.2) no scope creep, (FR-025.3) contracts read-only with escalation path, (FR-025.4) no self-review.

---

### SPEC-015 [PASS]

**FR**: FR-026 (Single invocation per WP)
**Classification**: Compliant
**Evidence**: Step 1 states "One `code-implementation` invocation handles all tasks in one WP" with topological sorting. Constraints section reiterates "Single Invocation Per WP" and prohibits requesting multiple invocations.
