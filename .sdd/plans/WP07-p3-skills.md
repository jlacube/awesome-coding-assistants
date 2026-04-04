---
lane: for_review
review_status: acknowledged
---

# WP07 - P3 Review Skills (review-performance, review-docs, review-deps)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P3 |
| Lane | for_review |
| Depends on | WP02 |
| Goal | Create three P3 review skills: review-performance (performance patterns), review-docs (documentation accuracy), and review-deps (dependency review), completing the full 8-skill review suite |
| Status | Complete |
| Independent Test | Install all 8 review skills. Invoke the coordinator on a WP. Verify: coordinator discovers and dispatches all 8 skills in canonical order, 8 findings files are created, aggregate verdict includes all dimensions in the statistics table |
| Parallelisable | Yes (with WP06; internal tasks for each skill are independent) |
| Prompt | `.sdd/plans/WP07-p3-skills.md` |

## Objective

Create three P3 review skills that complete the full 8-skill review suite. These skills add performance pattern detection, documentation accuracy verification, and dependency security/health review dimensions. Like P2 skills, they follow the common contract (FR-025-029) and are dispatched by the coordinator in positions 6-8 of the canonical order (FR-004). These are the lowest priority enhancements - the core review system works fully without them.

## Spec References

- Section 4.2 (FR-025 to FR-029) - Common skill contract
- Section 4.8 (FR-044 to FR-045) - Performance skill
- Section 4.9 (FR-046 to FR-047) - Documentation skill
- Section 4.10 (FR-048 to FR-049) - Dependencies skill
- Section 7.1 (Skill Findings File format)
- Section 7.5 (Skill File metadata)

## Tasks

### T07-01 - Create review-performance SKILL.md with frontmatter and purpose

- **Description**: Create the directory `.github/skills/review-performance/` and file `SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: Yes (independent of T07-03 through T07-07)
- **Acceptance criteria**:
  - [x] Directory `.github/skills/review-performance/` exists
  - [x] File `.github/skills/review-performance/SKILL.md` exists
  - [x] YAML frontmatter `name` is `review-performance`
  - [x] YAML frontmatter `description` explains: detects performance anti-patterns (N+1 queries, missing indexes, blocking in async, unbounded fetching, unnecessary computation, inefficient data structures, missing caching)
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Same SKILL.md pattern as P1/P2 skills:
    ```yaml
    ---
    name: review-performance
    description: "Performance review skill. Detects N+1 queries, missing indexes, blocking in async contexts, unbounded data fetching, unnecessary computation, inefficient data structures, and missing caching opportunities."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T07-02 - Write performance checklist, severity guidance, and output format

- **Description**: Write the 7-category performance checklist from FR-044, severity rules from FR-045, and the output format matching Section 7.1.
- **Spec refs**: FR-044 (7 check categories), FR-045 (severity rules), FR-027 (format), FR-028 (read-only), FR-029 (N/A)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Checklist covers all 7 categories: N+1 query patterns, missing database indexes, blocking in async contexts, unbounded data fetching, unnecessary computation in hot paths, inefficient data structures, missing caching
  - [x] Each category has at least 3 verifiable checklist items phrased as questions
  - [x] Default severity is WARN for all performance findings (FR-045)
  - [x] FAIL only when the issue violates a specific performance NFR from spec Section 10.1 (e.g., a query that would prevent meeting NFR-001's 30-minute review time)
  - [x] N/A with justification for categories not applicable (e.g., "No database access" for N+1 queries)
  - [x] Finding prefix is `PERF-` (e.g., `PERF-001`)
  - [x] Output format matches Section 7.1 with `files_reviewed` field
  - [x] Read-only constraint and complete example included
- **Test requirements**: BDD - performance scenario references
- **Depends on**: T07-01
- **Implementation Guidance**:
  - Performance findings are almost always WARN (advisory). FAIL is reserved for violations of explicit performance NFRs.
  - N+1 query detection: look for database queries inside loops. The subagent searches for query calls within for/while loops.
  - Blocking in async: look for synchronous I/O calls (file reads, HTTP requests, sleep) inside async functions.
  - Inefficient data structures: look for list linear searches (`if x in large_list`) that should use sets or dicts.
  - Example finding:
    ```markdown
    ### PERF-001 [WARN]
    - **Checklist item**: N+1 Query Pattern
    - **Requirement**: FR-044 category 1
    - **File**: src/api/orders.py#L30-L38
    - **Description**: Database query executed inside a loop iterating over user IDs.
    - **Expected**: Use a batch query or JOIN to load all related orders in one query.
    - **Evidence**: `db.query(Order).filter(user_id=uid)` called inside `for uid in user_ids`.
    ```

### T07-03 - Create review-docs SKILL.md with frontmatter and purpose

- **Description**: Create the directory `.github/skills/review-docs/` and file `SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: Yes (independent of T07-01, T07-02, T07-05 through T07-07)
- **Acceptance criteria**:
  - [x] Directory `.github/skills/review-docs/` exists
  - [x] File `.github/skills/review-docs/SKILL.md` exists
  - [x] YAML frontmatter `name` is `review-docs`
  - [x] YAML frontmatter `description` explains: compares documentation against actual implementation for accuracy, completeness, and staleness
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Same SKILL.md pattern:
    ```yaml
    ---
    name: review-docs
    description: "Documentation accuracy review skill. Compares .sdd/docs/ content against implementation for accuracy, completeness, and staleness."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T07-04 - Write documentation checklist, severity guidance, and output format

- **Description**: Write the 9-category documentation checklist from FR-046, severity rules from FR-047, and the output format.
- **Spec refs**: FR-046 (9 check categories), FR-047 (severity rules), FR-027 (format), FR-028 (read-only), FR-029 (N/A)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Checklist covers all 9 categories: architecture docs, API reference, configuration guide, data model docs, user guide, developer guide, deployment guide, staleness, completeness
  - [x] Each doc check verifies content against ACTUAL implementation (not just that the file exists)
  - [x] Architecture docs: module structure matches real directory layout
  - [x] API reference: endpoints, params, response schemas match actual code
  - [x] Configuration guide: env vars, defaults, options match actual code
  - [x] Staleness: no references to removed features, old APIs, deprecated behavior
  - [x] Completeness: all six standard doc files exist and are populated
  - [x] FAIL: missing or empty required doc files, inaccurate content
  - [x] WARN: minor omissions (missing one parameter in otherwise accurate doc)
  - [x] Finding prefix is `DOC-` (e.g., `DOC-001`)
  - [x] Output format matches Section 7.1 with `files_reviewed` field
- **Test requirements**: BDD - documentation scenario references
- **Depends on**: T07-03
- **Implementation Guidance**:
  - The 6 standard doc files referenced in FR-046: `architecture.md`, `api-reference.md`, `configuration-guide.md`, `user-guide.md`, `developer-guide.md`, `deployment-guide.md` - all under `.sdd/docs/`
  - Data model docs may be inline in the spec or in a separate doc file
  - Staleness detection: search doc files for references to functions/endpoints/env vars that no longer exist in the codebase
  - The subagent should read both the doc file AND the referenced code to compare them
  - Example finding:
    ```markdown
    ### DOC-001 [FAIL]
    - **Checklist item**: API Reference - Endpoint mismatch
    - **Requirement**: FR-046 category 2
    - **File**: .sdd/docs/api-reference.md#L45
    - **Description**: API reference documents GET /users/:id but implementation uses GET /api/v1/users/:id
    - **Expected**: API reference path should match actual route definition
    - **Evidence**: Route defined in src/routes/users.js#L12 as `/api/v1/users/:id`
    ```

### T07-05 - Create review-deps SKILL.md with frontmatter and purpose

- **Description**: Create the directory `.github/skills/review-deps/` and file `SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: Yes (independent of T07-01 through T07-04)
- **Acceptance criteria**:
  - [x] Directory `.github/skills/review-deps/` exists
  - [x] File `.github/skills/review-deps/SKILL.md` exists
  - [x] YAML frontmatter `name` is `review-deps`
  - [x] YAML frontmatter `description` explains: reviews dependencies for CVEs, abandonment, unnecessary packages, license compatibility, version pinning, and supply chain integrity
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Same SKILL.md pattern:
    ```yaml
    ---
    name: review-deps
    description: "Dependency review skill. Checks for known CVEs, abandoned packages, unnecessary dependencies, license compatibility, version pinning, and supply chain integrity."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T07-06 - Write dependencies checklist, severity guidance, and output format

- **Description**: Write the 6-category dependency checklist from FR-048, severity rules from FR-049, and the output format.
- **Spec refs**: FR-048 (6 check categories), FR-049 (severity rules), FR-027 (format), FR-028 (read-only), FR-029 (N/A)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Checklist covers all 6 categories: known CVEs, abandoned/unmaintained packages, unnecessary dependencies, license compatibility, version pinning, supply chain integrity
  - [x] CVE checks: use `#tool:web` to research CVEs for major dependencies against NVD/advisory databases
  - [x] Abandoned packages: flag deps with no commits in 12 months or explicitly archived
  - [x] Unnecessary deps: flag imported but unused deps, or deps duplicating runtime functionality
  - [x] License compatibility: verify dep licenses are compatible with project license
  - [x] Version pinning: verify exact pins or lockfile usage, flag floating ranges without lockfile
  - [x] Supply chain: verify lockfile exists with checksums, flag missing lockfiles
  - [x] FAIL: CVEs with CVSS >= 7.0 (High/Critical)
  - [x] WARN: low-severity CVEs (CVSS < 7.0), abandoned packages, license issues, missing lockfiles
  - [x] Finding prefix is `DEP-` (e.g., `DEP-001`)
  - [x] Output format matches Section 7.1 with `files_reviewed` field
- **Test requirements**: BDD - dependency scenario references
- **Depends on**: T07-05
- **Implementation Guidance**:
  - The subagent reads dependency manifests: `package.json`, `requirements.txt`, `pyproject.toml`, `Pipfile`, `Cargo.toml`, etc.
  - CVE lookup via web research: search NVD (nvd.nist.gov) or GitHub Advisory Database for each major dependency
  - CVSS score determines severity: >= 7.0 is FAIL, < 7.0 is WARN per FR-049
  - Lockfile detection: `package-lock.json`, `yarn.lock`, `Pipfile.lock`, `poetry.lock`, `Cargo.lock`
  - NFR-006 applies: only fetch from trusted URLs (nvd.nist.gov, npmjs.com, pypi.org, crates.io, GitHub advisories)
  - Example finding:
    ```markdown
    ### DEP-001 [FAIL]
    - **Checklist item**: Known CVEs - High severity
    - **Requirement**: FR-048 category 1, FR-049
    - **File**: package.json#L15
    - **Description**: lodash@4.17.20 has CVE-2021-23337 (CVSS 7.2 - command injection via template)
    - **Expected**: Upgrade to lodash@4.17.21 or later (patched)
    - **Evidence**: NVD lookup at https://nvd.nist.gov/vuln/detail/CVE-2021-23337
    ```

### T07-07 - Integration verification of all 8 skills with coordinator

- **Description**: Verify the complete 8-skill suite integrates correctly with the coordinator. Check end-to-end flow: all skills discovered, dispatched in canonical order, findings files created, aggregate verdict includes all dimensions.
- **Spec refs**: FR-003 (discovery), FR-004 (full 8-skill dispatch order), FR-010 (aggregation)
- **Parallel**: No (final task)
- **Acceptance criteria**:
  - [x] Coordinator discovers all 8 review skills via glob scan
  - [x] Dispatch order matches FR-004: review-spec, review-security, review-quality, review-tests, review-architecture, review-performance, review-docs, review-deps
  - [x] 8 findings files created in `.sdd/reviews/<WP-id>/` with correct naming and prefixes
  - [x] Finding prefixes across all skills are unique: SPEC-, SEC-, QUAL-, TEST-, ARCH-, PERF-, DOC-, DEP-
  - [x] Aggregate verdict statistics table includes all 8 skill dimensions
  - [x] Cross-correlation works across all dimensions (e.g., a security finding might correlate with a code quality finding)
- **Test requirements**: BDD - full suite discovery and dispatch
- **Depends on**: T07-02, T07-04, T07-06 (all three P3 skills must be complete)
- **Implementation Guidance**:
  - This is a verification task, not a code change. The Coder should:
    1. Ensure all 8 skills are installed in `.github/skills/review-*/`
    2. Invoke the coordinator on a test WP
    3. Verify the 8-skill dispatch, findings file creation, and aggregated report
  - The canonical order list in the coordinator (FR-004) already includes all 8 skills, so no coordinator change is needed
  - If any skill fails to dispatch, the coordinator records a WARN and continues (FR-007) - this is expected behavior, not a bug

## Implementation Notes

- This WP produces THREE files: `.github/skills/review-performance/SKILL.md`, `.github/skills/review-docs/SKILL.md`, `.github/skills/review-deps/SKILL.md`
- Each file follows the established structure: Purpose -> Checklist -> Severity Rules -> Output Format
- All three skills are independent of each other
- Target size: 100-150 lines each. P3 skills are the simplest (fewer categories, simpler severity rules)
- P3 directory creation is handled in this WP (not in WP01)
- The dependencies skill (review-deps) is the only skill that uses web research for CVE lookups. Like review-security, it must respect NFR-006 (trusted URLs only)

## Parallel Opportunities

- All three skills (T07-01/02, T07-03/04, T07-05/06) are fully parallel tracks
- T07-07 depends on all three tracks being complete

## Risks & Mitigations

- **Risk**: CVE lookups via web research are unreliable or rate-limited
  - Mitigation: The skill records WARN if web research fails ("Unable to verify CVEs via external source") and continues with static analysis per FR-036 pattern
- **Risk**: Documentation skill has nothing to review if `.sdd/docs/` is empty
  - Mitigation: Missing doc files produce FAIL findings (FR-047). The skill reports what is missing, which is useful feedback for the Coder.
- **Risk**: Dependency manifests vary widely across languages
  - Mitigation: Include a list of known dependency file patterns per language. If no recognized manifest is found, the skill marks dependency review as N/A.

## Review

### Round 1

| Field | Value |
|-------|-------|
| Reviewer | 5. Review Coordinator |
| Date | 2026-04-05 |
| Verdict | Changes Required |
| Round | 1 |

#### Aggregated Statistics

| Skill | PASS | WARN | FAIL | N/A |
|-------|------|------|------|-----|
| review-spec | 21 | 0 | 1 | 4 |
| review-security | 3 | 3 | 0 | 13 |
| review-quality | 6 | 0 | 0 | 2 |
| review-tests | 0 | 0 | 0 | 6 |
| review-architecture | 12 | 0 | 0 | 6 |
| review-performance | 0 | 0 | 0 | 7 |
| review-docs | 14 | 1 | 0 | 18 |
| review-deps | 0 | 0 | 0 | 6 |
| **Total** | **56** | **4** | **1** | **62** |

#### FAIL Findings

##### FB-01: SPEC-004 - review-deps omits spec file read (FR-026 step 2)
- [x] Resolved
- **Severity**: FAIL
- **Source**: review-spec SPEC-004
- **Requirement**: FR-026 (common execution steps, step 2: "Read the specification file to understand what was required")
- **File**: `.github/skills/review-deps/SKILL.md` lines 8-14
- **Description**: The review-deps input contract jumps from reading SKILL.md directly to identifying dependency manifest files, skipping the specification file read entirely. The spec may contain dependency-relevant requirements (NFRs about library versions, security constraints, license requirements) that the skill would miss.
- **Required fix**: Insert a new step 2 in the input contract: "Read the specification file to understand dependency-relevant requirements (NFRs, security constraints, license requirements)." Renumber subsequent steps 2-6 to 3-7. This matches the pattern used by review-performance and review-docs.

#### WARN Findings (Acknowledged)

##### SEC-002: P3 skills omit explicit NFR-004 static-analysis constraint
- **Severity**: WARN
- **Source**: review-security SEC-002
- **Description**: All three P3 skills lack the explicit "Do NOT execute code" constraint present in review-security. Defense-in-depth concern -- the skills don't actively encourage code execution but don't prohibit it either.
- **Action**: Recommended but not blocking. Consistent with WP06 SEC-002 WARN.

##### SEC-003: P3 skills omit NFR-005 secret non-reproduction constraint
- **Severity**: WARN
- **Source**: review-security SEC-003
- **Description**: None of the three P3 skills includes an explicit constraint against reproducing secret values in findings evidence. Consistent with WP06 SEC-002 pattern.
- **Action**: Recommended but not blocking.

##### SEC-004: review-deps NFR-006 constraint scoped too narrowly
- **Severity**: WARN
- **Source**: review-security SEC-004
- **Description**: review-deps lists trusted CVE sources for Category 1 but lacks the explicit "Do NOT fetch arbitrary URLs from the codebase" prohibition for other categories. Agent could follow URLs found in package manifests.
- **Action**: Recommended but not blocking. The trusted sources list provides partial coverage.

##### DOC-031: 3 of 6 standard doc files missing
- **Severity**: WARN
- **Source**: review-docs DOC-031
- **Description**: api-reference.md, configuration-guide.md, deployment-guide.md do not exist. Recurring WARN -- project type justifies absence.
- **Action**: Accepted.

#### Cross-Correlation Notes

- SPEC-004 (missing spec read) is independent of the security WARNs. No cross-correlation needed.
- SEC-002/SEC-003 are consistent with the same WARNs from WP06 review -- they reflect a project-wide pattern where only review-security includes explicit NFR constraints. This has been accepted as WARN across all WP reviews.
- No duplicate findings across skills.

#### Findings Directory

All individual skill findings are in `.sdd/reviews/WP07-p3-skills/`:
- `review-spec-findings.md`
- `review-security-findings.md`
- `review-quality-findings.md`
- `review-tests-findings.md`
- `review-architecture-findings.md`
- `review-performance-findings.md`
- `review-docs-findings.md`
- `review-deps-findings.md`

## Activity Log

- 2026-04-04T11:40:00Z - planner - lane=planned - Work package created
- 2026-04-04T17:00:00Z - coder - lane=doing - Starting WP07 implementation
- 2026-04-04T17:30:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T01:00:00Z - reviewer - lane=to_do - Round 1 review: Changes Required (1 FAIL: SPEC-004 missing spec read in review-deps)
- 2026-04-05T01:05:00Z - coder - lane=doing - Addressing reviewer feedback (FB-01)
- 2026-04-05T01:10:00Z - coder - lane=for_review - FB-01 resolved, resubmitted for review
