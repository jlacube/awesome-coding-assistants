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
9. **Coverage check** - Verifies 80% code, 90% branch coverage
10. **Handoff** - Sets lane to for_review and hands off to Reviewer

### Key Behaviors

- **No self-review**: The Coder does not assess its own work. The Reviewer is the sole quality gate.
- **Contract-first**: Implementation must match contract files exactly (signatures, types, fields, error codes).
- **Per-task commits**: Each task is committed individually with explicit `git add` file listing.
- **Debug retry**: If tests fail, the debug skill runs up to 3 times before escalating to the user.
