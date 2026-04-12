# Orchestrator Reference -- Detailed Algorithms

> This file contains detailed algorithms and edge-case handling for the Orchestrator.
> The Orchestrator loads this file on demand via `read_file` when executing
> WP selection, error recovery, or state file reconstruction.
> Do NOT embed this content in the agent file -- it is lazy-loaded to conserve context.

---

## Section 1: Dependency-Aware Topological Sort (WP Selection)

<!-- Spec refs: FR-040, FR-041, FR-042, FR-043; Section 6.5; Section 8.6 -->

The Orchestrator SHALL select the next WP for implementation using a topological sort of the dependency graph derived from `depends_on` frontmatter fields in WP files (FR-040). This replaces simple lowest-number-first ordering. When no WPs have `depends_on` fields, the sort degenerates to lowest-number-first (backward compatible).

Follow these steps in order:

### Step A: Read all WP files and build the dependency graph

1. Read all `.sdd/plans/WP*.md` files. For each WP, extract `lane` and `depends_on` from YAML frontmatter.
2. If a WP has no `depends_on` field or has `depends_on: []`, treat it as having no dependencies -- it is immediately eligible when its lane is `planned` (FR-043).
3. Build an adjacency list representing the dependency graph. Each edge goes from a dependency to the WP that depends on it (e.g., if WP03 depends on WP01, the edge is WP01 -> WP03).
4. Compute the in-degree for each WP (the number of dependencies it has).

### Step B: Validate dependency references

For each WP's `depends_on` list, verify that every referenced WP identifier corresponds to an existing WP file. If any reference is invalid:
- **Halt** with error E-051 (MISSING_DEPENDENCY): "{wp} depends on {dep} which does not exist."
- Do NOT proceed to WP selection.

### Step C: Detect circular dependencies

Use Kahn's algorithm to detect cycles (FR-042). After processing all nodes with in-degree 0 (Step D), if there are still unprocessed WPs remaining in the graph, a circular dependency exists.

If a cycle is detected:
- **Halt** with error E-050 (CIRCULAR_DEPENDENCY): "Circular dependency detected: WP-A -> WP-B -> ... -> WP-A. Cannot determine execution order."
- To identify the cycle: from the unprocessed WPs, pick one and follow its `depends_on` chain until a WP is visited twice. Report the cycle path.
- Do NOT proceed to WP selection.

Cycle detection runs BEFORE WP selection, not after.

### Step D: Perform topological sort (Kahn's algorithm)

1. Initialize a queue with all WPs that have in-degree 0 (no dependencies).
2. While the queue is not empty:
   a. Remove a WP from the queue.
   b. Add it to the sorted order.
   c. For each WP that depends on the removed WP, decrement its in-degree by 1. If the in-degree reaches 0, add it to the queue.
3. After the queue is empty, if the number of WPs in the sorted order is less than the total number of WPs, a cycle exists (see Step C).

This completes in O(V+E) time where V is the number of WPs and E is the number of dependency edges (NFR-002).

### Step E: Filter to eligible WPs

From the topological order, filter to WPs that meet ALL of these conditions:
1. The WP has `lane: planned` (not doing, done, for_review, to_do, or blocked).
2. ALL of the WP's dependencies have `lane: done`. A WP with no dependencies automatically satisfies this condition (FR-043).

### Step F: Apply tiebreaker and select

Among the eligible WPs, select the one with the **lowest WP number** (FR-041). This is deterministic -- the same input always produces the same output.

If two or more WPs can run in parallel and have no shared files, note this to the user but still execute sequentially (agents are single-threaded).

### Step G: Handle no-eligible-WP cases

If no WPs are eligible after filtering:

- **No WPs with `lane: planned` at all**: Normal state (all WPs in progress, under review, or done). No action needed.
- **WPs with `lane: planned` exist but ALL have unmet dependencies**: Report error E-052 (ALL_WPS_BLOCKED): "No WPs are ready. Blocked WPs: {list with unmet deps}."
  - This is a report, not a halt -- the Orchestrator continues processing other pipeline states.

---

## Section 2: Corrupted State File Recovery

When `.sdd/state.md` exists but has corrupted or invalid YAML frontmatter:

1. **Log a warning**: "State file at .sdd/state.md has corrupted YAML. Recreating from WP frontmatter ground truth."
2. **Scan WP frontmatter**: Read all `.sdd/plans/WP*.md` files and extract their `lane:` values to determine actual pipeline state.
3. **Reconstruct state**: Create a new state file replacing the corrupted one:
   - `pipeline_stage`: Derive from WP `lane` values using highest-urgency-first priority: `blocked` → escalate to user; `for_review` → `review`; `to_do` or `doing` or `planned` → `implementation`; all `done` → check documentation status. When multiple WPs have different lanes, the highest-urgency lane wins.
   - `current_wp`: Set to the lowest-numbered WP that is not `lane: done` (or null if all done)
   - `current_spec`: Derive from `.sdd/specs/` directory (the spec referenced by the current WP)
   - `last_agent`, `last_result`: Set to null (unknown after corruption)
   - `retry_count`: Set to 0
   - `error_log`: Set to empty array (history is lost)
   - `updated_at`: Set to current ISO 8601 timestamp
4. **Write the reconstructed state file**.
5. **Verify accuracy**: state file SHALL accurately reflect actual WP frontmatter `lane:` values.
6. **Proceed to Step 2** of the main workflow.

---

## Section 3: Detailed Error Handling

### Step 8b: On Failure -- Error Recording and Retry (FR-011)

When an agent invocation fails (agent reports error, produces no output, or times out):

1. **Record the failure** in `error_log` in `.sdd/state.md` with:
   - `agent`: Name of the failed agent (e.g., "4. Coder")
   - `wp`: WP identifier (e.g., "WP03") or null if not WP-scoped
   - `error_summary`: Human-readable summary, 1-500 characters. SHALL NOT contain full stack traces with sensitive paths.
   - `timestamp`: Current ISO 8601 timestamp
   - If `error_log` would exceed 50 entries, prune the oldest entry before adding the new one.

2. **Increment `retry_count`** in `.sdd/state.md`

3. **Evaluate retry threshold**:
   - If `retry_count` < 2: Retry the same agent with the same input. Log: "Retrying {agent} for {wp} (attempt {retry_count} of 2)". Return to Step 6 with the same agent and prompt.
   - If `retry_count` >= 2: **Escalate to user** (see Step 8c)

### Step 8c: Escalation on Max Retries (FR-011 step 4)

When `retry_count` >= 2, escalate to the user:

1. Present via `askQuestions`:
   - **Error summary**: What failed and why
   - **Agent name**: Which agent failed
   - **WP identifier**: Which WP was being processed
   - **Full error log**: ALL `error_log` entries for the current agent/WP
2. Wait for user response before continuing
3. When the user responds, reset `retry_count` to 0 and re-assess state (Step 3).

### Step 8d: On Escalation from Agent (FR-014)

When any agent reports an escalation (spec ambiguity, environment issue, unresolvable conflict):

1. Record the escalation: set `last_result: escalated`
2. Present to user via `askQuestions` with full context
3. Wait for user response -- pipeline halts until resolved
4. Follow Escalation Resolution Protocol (Step 8f)

### Step 8e: Review Failure Escalation (FR-012)

Track review cycles per WP using `review_cycles` frontmatter. When `review_cycles >= 2`:
- **Halt** -- do NOT continue retrying
- **Escalate to user** with all review feedback from all cycles
- Wait for user guidance

### Step 8f: Escalation Resolution Protocol (FR-015)

When user resolves an escalation:

1. Reset `last_result` to null and `retry_count` to 0
2. Re-read ALL state from disk (user may have modified files)
3. Use the Decision Table to determine the next action
4. Invoke the determined agent (may NOT be the same agent that escalated)

---

## Section 4: State Schema Definition

```yaml
---
pipeline_stage: "idle"       # One of: idle, ideation, specification, planning, implementation, review, documentation, complete
current_spec: null           # Path to active spec file or null
current_wp: null             # WP identifier (e.g., WP01) or null
last_agent: null             # Agent name or null
last_result: null            # success, failed, escalated, or null
retry_count: 0               # Integer >= 0
error_log: []                # Array of ErrorEntry (max 50)
updated_at: "ISO-8601"       # Timestamp of last update
---
```

### ErrorEntry Schema

| Field | Type | Required |
|-------|------|----------|
| agent | string | yes |
| wp | string or null | yes |
| error_summary | string (1-500 chars) | yes |
| timestamp | string (ISO 8601) | yes |
