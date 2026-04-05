---
lane: for_review
---

# WP26 - review-spec Contract-Aware Expansion

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/005-review-spec-completeness.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | Expand the existing review-spec skill to validate implementation code against formal contract files (interfaces, data schemas, API contracts, state machines, error catalogs) |
| Status | Not Started |
| Independent Test | Create a contract file defining `createUser(input: CreateUserInput)` and an implementation with `createUser(data: CreateUserInput)`. Run review-spec with contract files. Verify: a SPEC-CONTRACT finding for interface-mismatch reports parameter name mismatch ("input" vs "data") |
| Parallelisable | Yes (with WP25) |
| Prompt | `.sdd/plans/WP26-review-spec-contract-aware.md` |

## Objective

Expand the existing `.github/skills/review-spec/SKILL.md` to add contract-aware code review capabilities. The expanded skill validates implementation code against formal contract files in `.sdd/plans/contracts/<WP-slug>/` -- comparing function signatures, entity fields, API endpoints, state transitions, and error codes token-by-token. When contract files are absent (e.g., plans from Planner V1), the skill falls back to prose-only review. This ensures implementation drift from formal contracts is caught during review.

## Spec References

FR-015, FR-016, FR-017, FR-018, FR-019, Section 5 US-02, Section 7.2 (Contract Finding), Section 8.3 (Expanded Dispatch Prompt), Section 11.2 (BDD Scenarios for Contract-Aware Review)

## Tasks

### T26-01 - Preserve existing behavior and add contract-aware overview section

- **Description**: Add a new contract-aware review section to the existing `review-spec/SKILL.md`. The existing prose-based review behavior SHALL be fully preserved. The new section introduces the contract checking flow: discover contract files, load them, and run contract-specific checks after the prose checks.
- **Spec refs**: FR-015, FR-019, Section 8.3
- **Parallel**: No (foundation for all T26 tasks)
- **Acceptance criteria**:
  - [x] Existing review-spec behavior is fully preserved: FR classification, adherence checklist, stub detection, success criteria verification, severity rules, output format all remain unchanged (FR-015)
  - [x] A new section is added for contract-aware checks, clearly separated from the existing prose-based checks
  - [x] The section instructs: after completing prose-based review, check for contract files at `.sdd/plans/contracts/<WP-slug>/`
  - [x] If contract files exist, perform contract-based checks (FR-016)
  - [x] The dispatch prompt from spec Section 8.3 is compatible with the expanded skill
- **Test requirements**: BDD - Section 11.2 "Fallback to prose-only when no contracts" scenario
- **Depends on**: none
- **Implementation Guidance**:
  - Read the existing `review-spec/SKILL.md` first to understand the current structure
  - Add the contract-aware section AFTER all existing sections (do not interleave)
  - The contract files directory follows the pattern `.sdd/plans/contracts/<WP-slug>/` where WP-slug matches the WP file name (e.g., `WP25-review-spec-completeness`)
  - Contract file types to check for: `interfaces.<ext>`, `data-schemas.<ext>`, `api-contracts.<ext>`, `state-machines.<ext>`, `error-catalog.<ext>`

### T26-02 - Write interface contract check with token-level comparison

- **Description**: Write the checklist section for comparing implementation function/method signatures against contract files. Each parameter name, parameter type, and return type must match exactly.
- **Spec refs**: FR-016.1, FR-017
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Check instructs: every function/method signature in the implementation SHALL match the corresponding signature in `interfaces.<ext>` (FR-016.1)
  - [x] Comparison is token-by-token per FR-017: function/method names (exact match), parameter names (exact match), type annotations (exact match including generics and nullability)
  - [x] Mismatches are flagged as findings with category interface-mismatch
  - [x] BDD scenario covered: "Given contract defines createUser(input: CreateUserInput) and implementation has createUser(data: CreateUserInput), When review-spec runs with contract files, Then SPEC-CONTRACT finding for interface-mismatch is reported with expected 'input' and actual 'data'"
  - [x] Extra public functions not in the contract are flagged as MEDIUM findings (edge case from spec Section 5)
- **Test requirements**: BDD - Section 11.2 "Detect function signature mismatch" scenario
- **Depends on**: T26-01
- **Implementation Guidance**:
  - The comparison should be language-aware: TypeScript uses `:` for type annotations, Python uses `->` for return types and `:` for parameter types
  - Private/internal functions (prefixed with `_` in Python, not exported in TypeScript) are excluded from the check
  - The contract file `interfaces.<ext>` contains exported function/method signatures that the implementation must match
  - The finding format uses SPEC-CONTRACT-XXX prefix per FR-018

### T26-03 - Write data schema contract check with token-level comparison

- **Description**: Write the checklist section for comparing implementation entity/model classes against data schema contract files. Each field name, field type, constraint, and default must match exactly.
- **Spec refs**: FR-016.2, FR-017
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Check instructs: every entity/model class in the implementation SHALL match the corresponding definition in `data-schemas.<ext>` (FR-016.2)
  - [x] Comparison is token-by-token per FR-017: field names (exact match, case-sensitive), type annotations (exact match including generics and nullability)
  - [x] Mismatches are flagged as findings with category schema-mismatch
  - [x] BDD scenario covered: "Given contract defines User.email: string and implementation has User.emailAddress: string, When review-spec runs, Then SPEC-CONTRACT finding for schema-mismatch is reported"
- **Test requirements**: BDD - Section 11.2 "Detect field name mismatch" scenario
- **Depends on**: T26-01
- **Implementation Guidance**:
  - Data schema contract files define entity shapes: field names, types, optional/required, constraints
  - The check should compare each field in the contract against the corresponding implementation field
  - Missing fields in implementation (present in contract but absent in code) are HIGH findings
  - Extra fields in implementation (absent in contract) may be acceptable if they are computed/derived -- flag as MEDIUM

### T26-04 - Write API, state machine, and error catalog contract checks

- **Description**: Write the checklist sections for three contract checks: (1) API endpoints against `api-contracts.<ext>`, (2) state transitions against `state-machines.<ext>`, and (3) error codes against `error-catalog.<ext>`. Each uses token-level comparison.
- **Spec refs**: FR-016.3, FR-016.4, FR-016.5, FR-017
- **Parallel**: No
- **Acceptance criteria**:
  - [x] API contract check: every API endpoint in the implementation SHALL match `api-contracts.<ext>` -- method, path, request schema, response schema, error responses (FR-016.3)
  - [x] State machine check: every state transition in the implementation SHALL match `state-machines.<ext>` -- valid states, valid transitions, guards (FR-016.4)
  - [x] Error catalog check: every error code/message in the implementation SHALL match `error-catalog.<ext>` -- code value (exact match), HTTP status, message template (FR-016.5, FR-017)
  - [x] State enum values are compared with exact match per FR-017
  - [x] Error code string values are compared with exact match per FR-017
  - [x] BDD scenario covered: "Given contract error catalog has USR-001 and USR-002, and implementation only handles USR-001, When review-spec runs, Then SPEC-CONTRACT finding for error-mismatch is reported for USR-002"
- **Test requirements**: BDD - Section 11.2 "Detect missing error code" scenario
- **Depends on**: T26-01
- **Implementation Guidance**:
  - These three checks are grouped because they follow the same comparison pattern and are smaller in scope than interface/schema checks
  - Not every WP will have all contract types -- if a contract file does not exist, skip that check (no finding)
  - API contract comparison includes: HTTP method, URL path, request body fields, response body fields, and error response codes
  - State machine comparison: check that code enforces only valid transitions and does not allow transitions not in the contract

### T26-05 - Write contract mismatch finding format

- **Description**: Write the finding format section for contract-based findings (SPEC-CONTRACT-XXX), including all required fields: severity, category, contract_file, impl_file, expected, actual, recommendation.
- **Spec refs**: FR-018, Section 7.2 (Contract Finding)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Finding format matches FR-018 exactly: id (SPEC-CONTRACT-XXX), severity (HIGH), category (interface-mismatch | schema-mismatch | api-mismatch | state-mismatch | error-mismatch), contract_file, impl_file (path:line), expected, actual, recommendation
  - [x] All 5 mismatch categories are listed
  - [x] The expected field shows the contract definition; the actual field shows the implementation definition
  - [x] Default severity for mismatches is HIGH per FR-018
  - [x] Contract finding field constraints match Section 7.2: recommendation is 1-500 chars
- **Test requirements**: BDD - covered by scenarios in T26-02, T26-03, T26-04
- **Depends on**: T26-01
- **Implementation Guidance**:
  - Use the data model from `.sdd/specs/artifacts/005-review-spec-completeness/data-models.ts` as the canonical field reference for ContractFinding
  - The SPEC-CONTRACT-XXX IDs are sequential within a review run, starting from SPEC-CONTRACT-001
  - Contract findings are combined with prose-based findings in the final output -- the skill produces one unified findings file
  - Edge case: if a contract file has syntax errors, flag as HIGH finding and skip contract checks for that file (from spec edge cases section)

### T26-06 - Write fallback to prose-only handling

- **Description**: Write the fallback behavior section: when contract files do not exist for a WP, the skill falls back to prose-only review and notes the absence as an informational finding.
- **Spec refs**: FR-019
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] If contract files do not exist for the WP, the skill SHALL fall back to prose-only review (FR-019)
  - [x] The absence of contracts SHALL be noted as an informational (INFO) finding, not a failure
  - [x] Prose-only review uses the existing behavior (FR-015) with no degradation
  - [x] BDD scenario covered: "Given no contract files exist for the WP, When review-spec runs, Then it performs prose-only review and reports an INFO finding about missing contracts"
- **Test requirements**: BDD - Section 11.2 "Fallback to prose-only when no contracts" scenario
- **Depends on**: T26-01
- **Implementation Guidance**:
  - This is the graceful degradation path for WPs generated by Planner V1 (which does not produce contracts)
  - The INFO finding should mention: "No contract files found at .sdd/plans/contracts/<WP-slug>/. Falling back to prose-only review."
  - The skill should check for the contracts directory BEFORE attempting any contract checks
  - INFO findings do not affect the PASS/FAIL verdict

### T26-07 - Integration verification with Review Coordinator

- **Description**: Verify the expanded review-spec skill still integrates correctly with the Review Coordinator. Existing dispatch, discovery, and findings aggregation must work without coordinator changes.
- **Spec refs**: SC-003, FR-015, Section 9.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The expanded skill file is still discoverable via `review-*/SKILL.md` glob (SC-003)
  - [x] The coordinator's existing dispatch prompt for review-spec still works (backwards compatible)
  - [x] The expanded dispatch prompt from spec Section 8.3 provides contract file paths to the skill
  - [x] Combined findings (prose + contract) are aggregated correctly by the coordinator
  - [x] No Review Coordinator file changes are needed
- **Test requirements**: BDD - manual invocation via coordinator with and without contract files
- **Depends on**: T26-01 through T26-06
- **Implementation Guidance**:
  - The Review Coordinator at `.github/agents/reviewer.agent.md` dispatches review-spec as a subagent
  - The expanded dispatch prompt (spec Section 8.3) adds contract file paths -- the coordinator may need to pass the contracts directory, but since it already knows the WP slug, it can construct the path
  - Verify backwards compatibility: the skill should work when invoked with the OLD dispatch prompt (no contract paths) by falling back to prose-only
  - This is a verification task -- no new code needed, just confirmation

## Implementation Notes

- This WP modifies an existing file (`.github/skills/review-spec/SKILL.md`) rather than creating a new one. The existing 200-line file must be preserved and extended.
- The contract-aware checks are additive -- they run AFTER the existing prose-based checks.
- "Testing" means manually invoking the skill against a WP with known contract mismatches and verifying findings output.
- Tasks T26-02, T26-03, T26-04 write independent contract check sections. T26-05 and T26-06 are support sections. All depend on T26-01 (overview/structure).

## Parallel Opportunities

WP26 is parallelizable with WP25 since they modify different files. Within WP26, tasks T26-02 through T26-06 can conceptually be worked in parallel since they write independent sections, but T26-01 must come first. T26-07 must come last.

## Risks & Mitigations

- **Risk**: Expanding review-spec may break the existing prose-based review flow. **Mitigation**: T26-01 explicitly preserves all existing sections; T26-07 verifies backwards compatibility.
- **Risk**: Token-by-token comparison (FR-017) may be too strict, flagging equivalent but stylistically different code. **Mitigation**: The comparison targets structural elements (names, types) not formatting, reducing false positives.
- **Risk**: Contract file formats vary by target language, complicating instructions. **Mitigation**: The skill instructions are language-agnostic ("compare against `interfaces.<ext>`") and the subagent infers the language from the file extension.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T00:01:00Z - coder - T26-01 - completed - Added contract-aware overview section (Section 7) with discovery and loading
- 2026-04-06T00:02:00Z - coder - T26-02 - completed - Added interface contract check (Section 8) with token-level comparison
- 2026-04-06T00:03:00Z - coder - T26-03 - completed - Added data schema contract check (Section 9)
- 2026-04-06T00:04:00Z - coder - T26-04 - completed - Added API (Section 10), state machine (Section 11), error catalog (Section 12) checks
- 2026-04-06T00:05:00Z - coder - T26-05 - completed - Added contract finding format (Section 14)
- 2026-04-06T00:06:00Z - coder - T26-06 - completed - Added fallback to prose-only handling (Section 13)
- 2026-04-06T00:07:00Z - coder - T26-07 - completed - Verified integration with Review Coordinator (glob discovery, backwards compat)
- 2026-04-06T00:08:00Z - coder - lane=for_review - All tasks complete, tests passing, coverage met
