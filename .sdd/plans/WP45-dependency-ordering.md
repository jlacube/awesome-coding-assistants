---
lane: planned
---

# WP45 - Dependency-Aware WP Ordering

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P2 |
| Lane | planned |
| Depends on | WP40 |
| Goal | Replace the Orchestrator's WP-number-based selection with topological sort over the dependency graph |
| Status | Not Started |
| Independent Test | Create WP01 (depends_on: [WP02]) and WP02 (no deps), both with lane: planned. Verify the Orchestrator selects WP02 first despite WP01 having a lower number. |
| Parallelisable | Yes (with WP44) |
| Prompt | `.sdd/plans/WP45-dependency-ordering.md` |

## Objective

Update the Orchestrator agent instructions to select the next WP for implementation using topological sort over the `depends_on` dependency graph, with lowest WP number as tiebreaker. This replaces the current lowest-number-first approach, which fails when dependency order does not match WP numbering. The update also adds circular dependency detection and validation of dependency references.

## Spec References

FR-040, FR-041, FR-042, FR-043, Section 7.1 (depends_on field, state machine), Section 8.6 (Dependency Graph Interface), Section 6.5 (Dependency-Aware WP Selection flow), US-05

## Tasks

### T45-01 - Write topological sort algorithm in Orchestrator

- **Description**: Replace the "WP Selection Priority" section in `.github/agents/orchestrator.agent.md` with a topological sort algorithm that reads `depends_on` from all WP frontmatter and selects the next eligible WP.
- **Spec refs**: FR-040, Section 6.5 (steps 1-2, 5-8), Section 8.6
- **Parallel**: No (foundation for T45-02 through T45-04)
- **Acceptance criteria**:
  - [ ] Orchestrator SHALL select the next WP using topological sort of the dependency graph derived from `depends_on` frontmatter (FR-040)
  - [ ] Selected WP has all dependencies in `lane: done` or has no dependencies (FR-040)
  - [ ] Topological sort completes in O(V+E) time (NFR-002)
  - [ ] The algorithm description is clear enough for an LLM agent to follow step by step
- **Test requirements**: BDD (US-05 Scenario 1), E2E (Section 11.4 row 4)
- **Depends on**: none
- **Implementation Guidance**:
  - Use Kahn's algorithm (BFS-based topological sort) as it is easier to describe in natural language
  - Algorithm: 1) Read all WP files, extract depends_on. 2) Build adjacency list. 3) Compute in-degrees. 4) Process nodes with in-degree 0 whose dependencies are all done. 5) Select lowest-numbered from eligible set.
  - This replaces the existing "process WPs in numerical order" logic
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T45-02 - Add lowest-number tiebreaker

- **Description**: Specify that when multiple WPs have no unmet dependencies and are eligible (lane=planned), the Orchestrator selects the lowest WP number.
- **Spec refs**: FR-041, Section 6.5 (step 7)
- **Parallel**: No (part of T45-01's algorithm)
- **Acceptance criteria**:
  - [ ] Lowest-numbered WP among equally eligible WPs is selected as tiebreaker (FR-041)
  - [ ] Given WP03, WP01, WP02 all with no dependencies and lane: planned, the Orchestrator selects WP01 (US-05 Edge Case 1)
  - [ ] The tiebreaker is deterministic -- same input always produces same output
- **Test requirements**: BDD (US-05 Edge Case 1)
- **Depends on**: T45-01
- **Implementation Guidance**:
  - Add to the topological sort output: "Among WPs at the same topological level, select the one with the lowest number"
  - This preserves backward compatibility: when no dependencies exist, behavior matches the previous approach
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T45-03 - Add circular dependency detection

- **Description**: Add logic to the Orchestrator to detect circular dependencies in the dependency graph and halt with an error listing the cycle.
- **Spec refs**: FR-042, Section 6.5 (step 3), Section 8.6 (detect_cycles)
- **Parallel**: No (part of the dependency graph logic)
- **Acceptance criteria**:
  - [ ] Orchestrator SHALL detect circular dependencies and halt with cycle description (FR-042)
  - [ ] Error message format: "Circular dependency detected: WP-A -> WP-B -> ... -> WP-A" (FR-042)
  - [ ] Given WP01 depends on WP02 and WP02 depends on WP01, the Orchestrator halts with cycle description (US-05 Scenario 2)
  - [ ] Detection runs before WP selection, not after
- **Test requirements**: BDD (US-05 Scenario 2)
- **Depends on**: T45-01
- **Implementation Guidance**:
  - In Kahn's algorithm, if after processing all zero-in-degree nodes there are still unprocessed nodes, a cycle exists
  - Error E-050 (CIRCULAR_DEPENDENCY): halt with descriptive message
  - Also validate references: if depends_on references a WP that does not exist, halt with E-051 error (FR-042 implementation contract)
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T45-04 - Handle missing or empty depends_on

- **Description**: Document that WPs with no `depends_on` field or an empty array are treated as having no dependencies and are immediately eligible.
- **Spec refs**: FR-043, Section 6.5 (step 5-6)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] WPs with no `depends_on` field SHALL be treated as having no dependencies (FR-043)
  - [ ] WPs with `depends_on: []` SHALL be treated as having no dependencies (FR-043)
  - [ ] These WPs are eligible for implementation as soon as their lane is `planned`
- **Test requirements**: BDD (US-05 Edge Case 1)
- **Depends on**: T45-01
- **Implementation Guidance**:
  - This is the default case -- when `depends_on` is absent, the WP has in-degree 0 in the dependency graph
  - Ensure the algorithm handles both absent key and empty array identically
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T45-05 - Verify all-blocked reporting

- **Description**: Add logic to report when no WPs are eligible because all have unmet dependencies. Verify the Orchestrator produces a useful blocked-status report.
- **Spec refs**: FR-040, Section 8.6 (E-052)
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [ ] If no WP has all dependencies met and at least one WP has unmet dependencies, the Orchestrator SHALL report: "No WPs are ready. Blocked WPs: <list with unmet deps>" (FR-040)
  - [ ] The report lists each blocked WP and which dependencies are not yet done
  - [ ] Error E-052 (ALL_WPS_BLOCKED): report, do not halt
- **Test requirements**: content (review of Orchestrator instructions)
- **Depends on**: T45-01, T45-03
- **Implementation Guidance**:
  - This is distinct from circular dependency detection -- it handles the case where the graph is valid but no WP is currently ready
  - The Orchestrator should distinguish between "no WPs at all" (normal pipeline completion) and "WPs exist but all are blocked"
  - Files to modify: `.github/agents/orchestrator.agent.md`

## Implementation Notes

- All deliverables are markdown instruction updates to orchestrator.agent.md -- no executable code
- The topological sort is described in natural language for the LLM agent to follow -- not compiled code
- Backward compatibility: when no WPs have depends_on fields, the sort degenerates to lowest-number-first (same as current behavior)
- This WP modifies only orchestrator.agent.md, so it can run in parallel with WP44 (which modifies skill files)
- NFR-002 requires the sort to handle up to 100 WPs -- the algorithm description should be efficient

## Risks & Mitigations

- **Risk**: The topological sort instructions may be too complex for the LLM to follow reliably. **Mitigation**: Use simple step-by-step pseudocode rather than formal algorithm notation.
- **Risk**: Existing WP selection logic may be interleaved with other Orchestrator logic. **Mitigation**: Locate all WP selection references before editing.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
