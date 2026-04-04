---
lane: planned
---

# WP07 - P3 Review Skills (review-performance, review-docs, review-deps)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P3 |
| Lane | planned |
| Depends on | WP02 |
| Goal | Create three P3 review skills: review-performance (performance patterns), review-docs (documentation accuracy), and review-deps (dependency review), completing the full 8-skill review suite |
| Status | Not Started |
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
  - [ ] Directory `.github/skills/review-performance/` exists
  - [ ] File `.github/skills/review-performance/SKILL.md` exists
  - [ ] YAML frontmatter `name` is `review-performance`
  - [ ] YAML frontmatter `description` explains: detects performance anti-patterns (N+1 queries, missing indexes, blocking in async, unbounded fetching, unnecessary computation, inefficient data structures, missing caching)
  - [ ] Purpose section states the skill's role as subagent invoked by coordinator
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
  - [ ] Checklist covers all 7 categories: N+1 query patterns, missing database indexes, blocking in async contexts, unbounded data fetching, unnecessary computation in hot paths, inefficient data structures, missing caching
  - [ ] Each category has at least 3 verifiable checklist items phrased as questions
  - [ ] Default severity is WARN for all performance findings (FR-045)
  - [ ] FAIL only when the issue violates a specific performance NFR from spec Section 10.1 (e.g., a query that would prevent meeting NFR-001's 30-minute review time)
  - [ ] N/A with justification for categories not applicable (e.g., "No database access" for N+1 queries)
  - [ ] Finding prefix is `PERF-` (e.g., `PERF-001`)
  - [ ] Output format matches Section 7.1 with `files_reviewed` field
  - [ ] Read-only constraint and complete example included
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
  - [ ] Directory `.github/skills/review-docs/` exists
  - [ ] File `.github/skills/review-docs/SKILL.md` exists
  - [ ] YAML frontmatter `name` is `review-docs`
  - [ ] YAML frontmatter `description` explains: compares documentation against actual implementation for accuracy, completeness, and staleness
  - [ ] Purpose section states the skill's role as subagent invoked by coordinator
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
  - [ ] Checklist covers all 9 categories: architecture docs, API reference, configuration guide, data model docs, user guide, developer guide, deployment guide, staleness, completeness
  - [ ] Each doc check verifies content against ACTUAL implementation (not just that the file exists)
  - [ ] Architecture docs: module structure matches real directory layout
  - [ ] API reference: endpoints, params, response schemas match actual code
  - [ ] Configuration guide: env vars, defaults, options match actual code
  - [ ] Staleness: no references to removed features, old APIs, deprecated behavior
  - [ ] Completeness: all six standard doc files exist and are populated
  - [ ] FAIL: missing or empty required doc files, inaccurate content
  - [ ] WARN: minor omissions (missing one parameter in otherwise accurate doc)
  - [ ] Finding prefix is `DOC-` (e.g., `DOC-001`)
  - [ ] Output format matches Section 7.1 with `files_reviewed` field
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
  - [ ] Directory `.github/skills/review-deps/` exists
  - [ ] File `.github/skills/review-deps/SKILL.md` exists
  - [ ] YAML frontmatter `name` is `review-deps`
  - [ ] YAML frontmatter `description` explains: reviews dependencies for CVEs, abandonment, unnecessary packages, license compatibility, version pinning, and supply chain integrity
  - [ ] Purpose section states the skill's role as subagent invoked by coordinator
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
  - [ ] Checklist covers all 6 categories: known CVEs, abandoned/unmaintained packages, unnecessary dependencies, license compatibility, version pinning, supply chain integrity
  - [ ] CVE checks: use `#tool:web` to research CVEs for major dependencies against NVD/advisory databases
  - [ ] Abandoned packages: flag deps with no commits in 12 months or explicitly archived
  - [ ] Unnecessary deps: flag imported but unused deps, or deps duplicating runtime functionality
  - [ ] License compatibility: verify dep licenses are compatible with project license
  - [ ] Version pinning: verify exact pins or lockfile usage, flag floating ranges without lockfile
  - [ ] Supply chain: verify lockfile exists with checksums, flag missing lockfiles
  - [ ] FAIL: CVEs with CVSS >= 7.0 (High/Critical)
  - [ ] WARN: low-severity CVEs (CVSS < 7.0), abandoned packages, license issues, missing lockfiles
  - [ ] Finding prefix is `DEP-` (e.g., `DEP-001`)
  - [ ] Output format matches Section 7.1 with `files_reviewed` field
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
  - [ ] Coordinator discovers all 8 review skills via glob scan
  - [ ] Dispatch order matches FR-004: review-spec, review-security, review-quality, review-tests, review-architecture, review-performance, review-docs, review-deps
  - [ ] 8 findings files created in `.sdd/reviews/<WP-id>/` with correct naming and prefixes
  - [ ] Finding prefixes across all skills are unique: SPEC-, SEC-, QUAL-, TEST-, ARCH-, PERF-, DOC-, DEP-
  - [ ] Aggregate verdict statistics table includes all 8 skill dimensions
  - [ ] Cross-correlation works across all dimensions (e.g., a security finding might correlate with a code quality finding)
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

## Activity Log

- 2026-04-04T11:40:00Z - planner - lane=planned - Work package created
