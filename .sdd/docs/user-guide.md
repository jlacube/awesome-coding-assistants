# User Guide

## Overview

The Review Coordinator reviews implemented work packages against their specifications, plans, and documentation. It dispatches specialized review skills, aggregates findings, and produces a verdict with actionable feedback.

## Invoking the Coordinator

### Direct invocation

Type in VS Code Copilot Chat:

```
@review-coordinator WP01
```

Replace `WP01` with the work package ID you want reviewed.

### Without a WP ID

```
@review-coordinator
```

The coordinator scans for work packages with `lane: for_review`. If multiple are found, it asks you to choose.

### Via Orchestrator

The Orchestrator agent automatically invokes the coordinator when a WP reaches `lane: for_review`.

## What Happens During a Review

1. **Scope selection** - Identifies the WP to review
2. **Artifact loading** - Loads the WP plan, spec, ideation brief, and plan index
3. **Process compliance** - Checks acceptance criteria, Activity Log, commit granularity
4. **Encoding check** - Scans for prohibited Unicode characters (em dashes, smart quotes, etc.)
5. **Skill discovery** - Finds all installed review skills
6. **Skill dispatch** - Runs each skill sequentially as a subagent
7. **Aggregation** - Reads all findings, merges duplicates, flags conflicts
8. **Verdict** - Determines the review outcome
9. **Report** - Writes the review summary to the WP file
10. **Commit** - Commits all review artifacts

## Understanding Verdicts

| Verdict | Meaning | WP Lane |
|---------|---------|---------|
| **Approved** | Zero FAILs, zero WARNs | `done` |
| **Approved with Findings** | Zero FAILs, one or more WARNs | `done` |
| **Changes Required** | One or more FAILs | `to_do` |

## Understanding FB-XX Items

When the verdict is "Changes Required", the review report contains **FB-XX** items -- actionable feedback entries that must be resolved before re-review.

Each FB-XX item includes:
- **Dimension tag** (e.g., `[spec-adherence]`, `[security]`)
- **Requirement reference** (e.g., FR-002)
- **File path and line** where the issue was found
- **Expected fix** describing what needs to change
- **Source skills** identifying which review skill(s) flagged it

Example:
```
- [ ] **FB-01**: [spec-adherence] FR-002 Deviating - Coordinator treats
  ideation brief as optional.
  File: .github/agents/review-coordinator.agent.md#L66-L68.
  Expected: Halt for missing brief per FR-002.
  Source skills: review-spec (SPEC-002)
```

## Warnings

WARNs are informational findings that do not block approval. They appear under `### Warnings` in the review report and are not assigned FB-XX numbers.

## Re-Review Workflow

After fixing all FB-XX items:

1. The Coder sets `lane: for_review` on the WP
2. The coordinator is invoked again
3. On re-review, only relevant skills are re-dispatched:
   - Skills that previously FAILed
   - Skills whose reviewed files were modified
4. Findings from non-re-dispatched skills are preserved

## Stalled Reviews

If the same FB-XX items remain unresolved after 3 review rounds, the coordinator sets `lane: blocked` and escalates to the user.

## Handoff Buttons

After delivering a verdict, the coordinator offers handoff buttons:

| Button | Target | When to use |
|--------|--------|-------------|
| **Fix Findings** | Coder | Send FB-XX items for remediation (auto-sent on Changes Required) |
| **Update Specification** | Spec Architect | Spec gaps found during review |
| **Revise Plan** | Planner | Plan-level issues found during review |

## Central Enum Registry (WP40)

All pipeline-wide enumeration values are defined in a single registry file at `.github/schemas/enums.yaml`. This file is the authoritative source for valid values of `lane`, `spec_status`, `pipeline_stage`, and `review_status`.

**What it does**: Provides a single source of truth for enum values used across all agents, eliminating inconsistencies from scattered inline value lists.

**When to use it**: Reference this file whenever you need to know the valid values for WP lane states, spec statuses, pipeline stages, or review statuses.

### Enum Groups

| Group | Valid Values |
|-------|-------------|
| `lane` | planned, doing, for_review, to_do, done, blocked |
| `spec_status` | Draft, Validated, Approved |
| `pipeline_stage` | idle, ideation, specification, planning, implementation, review, documentation, complete |
| `review_status` | pending, has_feedback, acknowledged, approved |

## Canonical Activity Log Format (WP40)

All agents write Activity Log entries in WP files using a standardized format:

```
<ISO-8601-timestamp> - <agent-name> - <action> - <details>
```

Fields are separated by ` - ` (space-hyphen-space). Agent names use lowercase with hyphens (e.g., `coder`, `review-coordinator`, `docs-agent`).

The canonical format template is stored in `.github/schemas/enums.yaml` under `conventions.activity_log_format`.

## Troubleshooting

### Enum Registry Errors (WP40)

#### "Enum registry not found at .github/schemas/enums.yaml"

**Cause**: The central enum registry file is missing from the workspace.

**Resolution**:
1. Verify the file exists at `.github/schemas/enums.yaml`
2. If deleted accidentally, restore it from version control

**Prevention**: Do not delete or move `.github/schemas/enums.yaml` -- all agents depend on it.

#### "Enum group '<name>' not found in enums.yaml"

**Cause**: A required enum group (`lane`, `spec_status`, `pipeline_stage`, or `review_status`) is missing from the registry file.

**Resolution**:
1. Open `.github/schemas/enums.yaml`
2. Verify all four enum groups are present with their required values
3. Fix any YAML formatting issues

**Prevention**: Do not remove enum groups from the registry. Adding new values requires a spec change.

## Where to Find Detailed Findings

- **Summary**: In the WP file under `## Review`
- **Detailed per-skill findings**: In `.sdd/reviews/<WP-id>/` (one file per skill)
- **Review patterns**: In `.sdd/reviews/review-patterns.md`

---

## Coder Coordinator

The Coder Coordinator implements work packages by dispatching coding skills sequentially. It does not write code itself.

### Invoking the Coder

Type in VS Code Copilot Chat:

```
@coder WP03
```

Or without a WP ID to be prompted:

```
@coder
```

### What Happens During Implementation

1. **WP selection** - Identifies the WP to implement
2. **Artifact loading** - Reads WP, spec, contracts, patterns, AGENTS.md
3. **Dependency check** - Verifies prior WPs are done
4. **Contract validation** - Verifies referenced contract files exist
5. **Patterns consumption** - Reads code-patterns.md for mistakes to avoid
6. **Skill discovery** - Finds all installed coding skills
7. **Skill dispatch** - Runs skills sequentially: env-setup, implementation, unit-tests, integration-tests
8. **Debug (if needed)** - Up to 3 attempts to fix failing tests
9. **Coverage check** - Verifies coverage meets WP thresholds (configurable, defaults 80% code / 90% branch)
10. **Handoff** - Sets lane to for_review and hands off to Reviewer

### Key Behaviors

- **No self-review**: The Coder does not assess its own work. The Reviewer is the sole quality gate.
- **Contract-first**: Implementation must match contract files exactly (signatures, types, fields, error codes).
- **Per-task commits**: Each task is committed individually with explicit `git add` file listing.
- **Debug retry**: If tests fail, the debug skill runs up to 3 times before escalating to the user.

## Configurable Coverage Thresholds (WP44)

Coverage thresholds can be overridden per-WP using optional YAML frontmatter fields. This lets pipeline operators set project-appropriate coverage targets without modifying skill instructions.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `coverage_code` | Integer (0-100) | 80 | Minimum code coverage percentage |
| `coverage_branch` | Integer (0-100) | 90 | Minimum branch coverage percentage |

### Usage

Add fields to the WP file's YAML frontmatter:

```yaml
---
lane: planned
coverage_code: 60
coverage_branch: 70
---
```

### Behaviors

- **Absent fields**: Skills use the defaults (80% code, 90% branch)
- **Independent fields**: Specifying one does not require specifying the other. For example, `coverage_code: 60` alone uses 60% code and the default 90% branch.
- **Zero is valid**: Setting a threshold to 0 disables coverage enforcement for that dimension (useful for prototyping WPs)
- **Invalid values**: If a value is out of range (not 0-100) or not an integer, the skill halts with an error

### Affected Skills

The following skills read these fields:
- **code-unit-tests** -- uses thresholds for coverage enforcement
- **code-env-setup** -- uses thresholds when configuring coverage tooling
- **spec-test-strategy** -- references configurable thresholds in test requirements

## Mid-Cycle Pattern Propagation (WP47)

When the Review Coordinator discovers a new pattern during a review, that pattern can take effect within the same pipeline run -- without restarting coordinators.

### How It Works

Each domain pattern file (`spec-patterns.md`, `plan-patterns.md`, `code-patterns.md`, `doc-patterns.md`) has a `patterns_version` integer in its YAML frontmatter. Before each skill dispatch, coordinator agents check whether `patterns_version` has changed since they last read the file. If it has, the coordinator reloads the patterns and uses the updated content for subsequent skills.

### What You See

- **Automatic**: Pattern propagation is transparent. You do not need to restart or re-invoke coordinators.
- **Review adds a pattern**: When the Review Coordinator adds, modifies, or retires a pattern, it increments `patterns_version` in that file automatically.
- **Next skill picks it up**: The next skill dispatch by any coordinator detects the version change and reloads the patterns.

### Error Handling

- If a patterns file becomes unreadable during a re-check, the coordinator uses the last successfully cached patterns and logs a warning.
- If a patterns file has no YAML frontmatter, `patterns_version` is treated as 0, causing a reload on every dispatch (safe default).
