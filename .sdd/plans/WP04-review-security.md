---
lane: to_do
review_status: has_feedback
---

# WP04 - Security Review Skill (review-security)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP02 |
| Goal | Create the review-security skill that audits implementation code against all 14 OWASP Secure Coding Practices categories, replacing the current 3-bullet security check with expert-level depth |
| Status | Not Started |
| Independent Test | Install the review-security skill and invoke the coordinator on a WP with known security issues (SQL injection, hardcoded secret, missing input validation). Verify: findings file contains FAIL entries referencing specific OWASP categories with file paths, line ranges, and code evidence |
| Parallelisable | Yes (with WP03, WP05) |
| Prompt | `.sdd/plans/WP04-review-security.md` |

## Objective

Create `.github/skills/review-security/SKILL.md` - the security review skill. This is the primary gap identified in the brainstorming brief: the current reviewer has a 3-bullet security summary while OWASP defines 100+ checklist items across 14 categories. This skill provides expert-level security review by evaluating every applicable OWASP Secure Coding Practices checklist item against the implementation, cross-referencing against the spec's security requirements, and using web research to verify unfamiliar patterns. It directly addresses SC-002 (14 OWASP categories covered).

## Spec References

- Section 4.2 (FR-025 to FR-029) - Common skill contract
- Section 4.4 (FR-034 to FR-036) - Security skill requirements
- Section 7.1 (Skill Findings File format)
- Section 7.5 (Skill File metadata)
- Section 10.2 (Security NFRs: NFR-004, NFR-005, NFR-006)
- Section 11.2 (BDD scenarios for security skill)
- Section 17 (OWASP SCP reference link)

## Tasks

### T04-01 - Create SKILL.md with frontmatter and purpose

- **Description**: Create the file `.github/skills/review-security/SKILL.md` with YAML frontmatter and a purpose statement explaining the skill's role as the OWASP-based security auditor.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: No (foundation for all T04 tasks)
- **Acceptance criteria**:
  - [x] File exists at `.github/skills/review-security/SKILL.md`
  - [x] YAML frontmatter `name` is `review-security`
  - [x] YAML frontmatter `description` explains: audits code against 14 OWASP Secure Coding Practices categories
  - [x] Purpose section states: invoked by coordinator as subagent, reads SKILL.md + spec, discovers code, evaluates OWASP checklist, writes findings
  - [x] `.gitkeep` file removed from `.github/skills/review-security/` (replaced by SKILL.md)
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the same frontmatter pattern established by WP03 (review-spec):
    ```yaml
    ---
    name: review-security
    description: "Security review skill. Audits implementation against all 14 OWASP Secure Coding Practices categories."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```
  - Include a note that this skill uses web research (`#tool:web`) per FR-036

### T04-02 - Write OWASP categories 1-7 checklist

- **Description**: Write the checklist items for the first 7 OWASP SCP categories: Input Validation, Output Encoding, Authentication and Password Management, Session Management, Access Control, Cryptographic Practices, Error Handling and Logging.
- **Spec refs**: FR-034 (14 OWASP categories - first half), Section 17 (OWASP SCP reference)
- **Parallel**: Yes (can be written alongside T04-03)
- **Acceptance criteria**:
  - [x] Input Validation checklist includes: server-side validation, allow-list approach, data type/range/length checks, centralized validation, canonicalization, rejection on failure
  - [x] Output Encoding checklist includes: server-side encoding, context-appropriate encoding, sanitization for SQL/XML/LDAP/OS commands
  - [x] Authentication checklist includes: auth for non-public resources, secure credential storage (salted hashes), fail-secure, no credential leakage in errors, account lockout, MFA for sensitive ops
  - [x] Session Management checklist includes: server-side creation, sufficient randomness, inactivity timeout, new ID on re-auth, no IDs in URLs/logs, HttpOnly+Secure flags
  - [x] Access Control checklist includes: centralized authorization, fail-secure, enforced on every request, least privilege, RBAC/ABAC enforcement
  - [x] Cryptographic Practices checklist includes: approved algorithms only, secure random generation, proper key management
  - [x] Error Handling/Logging checklist includes: no sensitive data in errors, generic messages to users, centralized logging, log security events, no sensitive data in logs
  - [x] Each checklist item is phrased as a verifiable check (not a general principle)
- **Test requirements**: BDD - Section 11.2 "SQL injection detected", "Hardcoded secret detected" scenarios
- **Depends on**: T04-01
- **Implementation Guidance**:
  - Source: OWASP Secure Coding Practices Quick Reference Guide v2.1
    - Full checklist: https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/stable-en/02-checklist/05-checklist.html
  - Each category should have 4-8 checklist items (the most actionable ones from OWASP)
  - Items should be phrased as questions the subagent can answer by reading code: "Does the code validate all inputs server-side?" rather than "Input validation should be done server-side"
  - Categories where no relevant code exists should be marked N/A with justification per FR-029
  - Important: the skill MUST NOT execute any code (NFR-004) - review is static analysis only

### T04-03 - Write OWASP categories 8-14 checklist

- **Description**: Write the checklist items for the remaining 7 OWASP SCP categories: Data Protection, Communication Security, System Configuration, Database Security, File Management, Memory Management, General Coding Practices.
- **Spec refs**: FR-034 (14 OWASP categories - second half)
- **Parallel**: Yes (can be written alongside T04-02)
- **Acceptance criteria**:
  - [x] Data Protection checklist includes: least privilege data access, encrypted sensitive data at rest, no secrets in source code, no sensitive data in GET params, cache control for sensitive pages
  - [x] Communication Security checklist includes: TLS for sensitive data, valid certificates, no insecure fallback, character encoding specified
  - [x] System Configuration checklist includes: latest versions, patches applied, unnecessary functionality removed, test code removed from prod, HTTP methods restricted, security headers
  - [x] Database Security checklist includes: parameterized queries, least-privilege DB access, no hardcoded connection strings, default credentials changed
  - [x] File Management checklist includes: no user data in dynamic includes, auth before upload, file type validation by headers, upload directory execution disabled
  - [x] Memory Management checklist includes: buffer size checks, resource cleanup, no known vulnerable functions
  - [x] General Coding Practices checklist includes: no direct OS commands from user input, checksums for integrity, locking for race conditions, no dynamic code execution from user data
  - [x] Each category is clearly labeled and organized for easy subagent navigation
- **Test requirements**: BDD - Section 11.2 security scenarios
- **Depends on**: T04-01
- **Implementation Guidance**:
  - Same source and format as T04-02
  - Memory Management and some File Management items will frequently be N/A for high-level languages (Python, JavaScript) - include guidance that the subagent should mark these N/A with justification rather than skip them silently
  - Database Security items are critical - SQL injection is the most commonly tested vulnerability. Ensure the parameterized queries check is thorough.
  - Data Protection "no secrets in source code" check: instruct the subagent to look for patterns like API keys, passwords, tokens as string literals. But per NFR-005, the findings file MUST NOT reproduce the actual secret value - reference the file and line only.

### T04-04 - Write spec security cross-reference instructions

- **Description**: Write instructions for the skill to cross-reference its findings against the spec's security requirements (Section 10.2 NFRs) to verify all specified security controls are implemented.
- **Spec refs**: FR-035 (spec security cross-reference)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Skill reads the spec's Section 10.2 (Security NFRs) to identify specified security controls
  - [x] Each specified security control is verified as implemented in code
  - [x] Missing security controls produce FAIL findings with the specific NFR reference
  - [x] Security controls found in code but not in spec are noted (informational, not FAIL)
- **Test requirements**: BDD - Section 11.2 security scenarios
- **Depends on**: T04-02, T04-03 (must have OWASP checklist as context)
- **Implementation Guidance**:
  - The spec's Section 10.2 defines project-specific security requirements (e.g., NFR-004: no code execution, NFR-005: no secret reproduction, NFR-006: trusted URLs only)
  - This cross-reference catches cases where the OWASP checklist alone might miss project-specific security needs
  - The skill should read the spec's security section and create additional checklist items for any project-specific controls not already covered by OWASP categories

### T04-05 - Write web research instructions

- **Description**: Write instructions for the skill to use web research to verify unfamiliar security patterns against current OWASP guidelines, framework-specific docs, or CVE databases.
- **Spec refs**: FR-036 (web research for unfamiliar patterns), NFR-006 (trusted URLs only)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Skill is instructed to use `#tool:web` when encountering unfamiliar security patterns in code
  - [x] Web research targets are restricted to well-known security resources: OWASP, NVD (nvd.nist.gov), framework-specific security docs (e.g., Django security, Express security)
  - [x] Skill SHALL NOT fetch arbitrary URLs from the codebase (NFR-006)
  - [x] If web research fails, skill records WARN: "Unable to verify against external source" and continues with checklist-based review
  - [x] Web research findings are cited with source URL in the evidence field
- **Test requirements**: none (web research is opportunistic)
- **Depends on**: T04-01
- **Implementation Guidance**:
  - Provide a list of trusted domains the subagent may fetch from:
    - `owasp.org` (OWASP guidelines and checklists)
    - `nvd.nist.gov` (National Vulnerability Database, CVE lookup)
    - Framework-specific: `docs.djangoproject.com/en/*/topics/security/`, `expressjs.com/en/advanced/best-practice-security.html`, etc.
  - Use case: the subagent encounters a custom authentication pattern it does not recognize. Web research can verify whether that pattern is secure per current best practices.
  - The web research should SUPPLEMENT the OWASP checklist, not replace it. The checklist is the primary review mechanism.

### T04-06 - Write severity guidance, N/A handling, and output format

- **Description**: Write the severity rules, N/A handling guidance per FR-029, and the output format instructions matching Section 7.1. Include the secret non-reproduction constraint from NFR-005.
- **Spec refs**: FR-027 (findings format), FR-028 (read-only), FR-029 (N/A handling), FR-034 (severity implied by OWASP), NFR-005 (no secret reproduction)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] FAIL: violations of OWASP checklist items that represent exploitable vulnerabilities (SQL injection, XSS, hardcoded secrets, missing auth, etc.)
  - [x] WARN: non-critical security concerns (missing cache control headers, informational findings, best practices not followed but not exploitable)
  - [x] PASS: checklist items verified as correctly implemented
  - [x] N/A: checklist items not applicable to the codebase (with justification, e.g., "No database access in this WP")
  - [x] NFR-005 enforced: findings MUST NOT reproduce actual secret values found in code. Reference file and line only.
  - [x] Output format matches Section 7.1 exactly: YAML frontmatter + markdown findings
  - [x] Finding prefix is `SEC-` (e.g., `SEC-001`, `SEC-002`)
  - [x] `files_reviewed` in frontmatter lists all files the skill evaluated
  - [x] Explicit instruction: "Do NOT modify any source code, WP file, or spec file" (FR-028)
  - [x] NFR-004 enforced: "Do NOT execute any code from the codebase - review is static analysis only"
- **Test requirements**: BDD - Section 11.2 "SQL injection detected", "Non-applicable category skipped", "Hardcoded secret detected" scenarios
- **Depends on**: T04-02, T04-03, T04-04 (must have all checklist items defined)
- **Implementation Guidance**:
  - Severity guidance should be organized by OWASP category so the subagent can quickly determine severity
  - Example severity mapping:
    - SQL injection -> FAIL (exploitable)
    - Hardcoded secret -> FAIL (exploitable)
    - Missing input validation -> FAIL (exploitable)
    - Missing cache control -> WARN (not directly exploitable)
    - Missing HSTS header -> WARN (best practice)
  - Secret non-reproduction example:
    ```markdown
    ### SEC-005 [FAIL]
    - **Checklist item**: Data Protection - No secrets in source code
    - **Requirement**: NFR-005, OWASP SCP 2.8
    - **File**: src/config.py#L15
    - **Description**: API key found as string literal in source code.
    - **Expected**: Use environment variables or secret management.
    - **Evidence**: String literal assigned to API_KEY variable (value not reproduced per NFR-005)
    ```
  - Include complete example findings file showing proper format for all severity levels

## Implementation Notes

- This is a SINGLE file: `.github/skills/review-security/SKILL.md`
- All tasks contribute sections to this one file
- File structure: Purpose -> OWASP Checklist (14 categories) -> Spec Cross-Reference -> Web Research -> Severity Rules -> Output Format
- Target size: 200-280 lines. This is the largest skill due to the 14-category checklist. Must stay under 300 lines (C-002).
- The 14 OWASP categories are the skill's core value - they replace the current 3-bullet security check and deliver SC-002
- Many categories will be N/A for typical markdown/agent projects - the N/A handling with justification is essential to avoid false negatives

## Parallel Opportunities

- T04-02 and T04-03 can run concurrently (two halves of the OWASP checklist)
- T04-05 is independent and can run alongside T04-04
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: 14-category OWASP checklist pushes file over 300-line limit (C-002)
  - Mitigation: Use concise checklist format (one line per item). Group related items. Prioritize the most actionable items per category (4-6 per category). Target 200-280 lines.
- **Risk**: Subagent marks everything N/A for non-web-application codebases
  - Mitigation: Include guidance that even agent/skill markdown files have security considerations (e.g., prompt injection, tool restrictions, secret handling in instructions)
- **Risk**: Web research causes subagent to fetch untrusted URLs from code
  - Mitigation: Explicitly restrict web research to trusted domains listed in T04-05. Include NFR-006 constraint prominently.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-04T23:00:00Z
> **Verdict**: Changes Required
> **Skills dispatched**: review-spec (FAIL), review-security (WARN), review-quality (PASS), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (WARN), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 6 tasks have acceptance criteria checked off
- [PASS] Activity Log: Consistent lane transitions planned -> doing -> for_review
- [WARN] Commit granularity: Single commit (ddc0804) for entire WP rather than per-task commits
- [PASS] Encoding: No prohibited Unicode characters found

### Review Feedback

> Implementers: address every FB-XX item before returning for re-review.

- [ ] **FB-01**: [spec-adherence] FR-034 Partial - 4 OWASP checklist items specified in FR-034 are missing from the implementation. Add the following items to their respective categories in `.github/skills/review-security/SKILL.md`:
  1. Category 6 (Cryptographic Practices): "FIPS 140-2 compliance where required"
  2. Category 11 (Database Security): "Stored procedures for data abstraction"
  3. Category 12 (File Management): "No user data in redirects" (also in T04-03 acceptance criteria)
  4. Category 13 (Memory Management): "Null termination handling"
  File: .github/skills/review-security/SKILL.md#L63-L113. Expected: All checklist items enumerated in FR-034 for each of the 14 categories are present.
  Source skills: review-spec (SPEC-006), review-security (SEC-017, SEC-018, SEC-019, SEC-020)

### Warnings
- [WARN] PROC-003: Single commit for entire WP. Consider per-task commits for easier review and bisection.
- [WARN] DOC-020: 3 of 6 standard doc files missing (api-reference.md, configuration-guide.md, deployment-guide.md). All cover domains not applicable to this project. (review-docs DOC-020)

### Cross-Correlation Notes
- Merged duplicate: review-spec SPEC-006 (FAIL: FR-034 Partial) and review-security SEC-017/SEC-018/SEC-019/SEC-020 (4 WARNs) reference the same 4 missing OWASP checklist items. Merged into composite FB-01 with FAIL severity (most severe). 4 WARNs absorbed.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 16 | 0 | 1 |
| review-security | 7 | 4 | 0 |
| review-quality | 6 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 8 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 15 | 1 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **55** | **6** | **1** |

## Activity Log

- 2026-04-04T11:25:00Z - planner - lane=planned - Work package created
- 2026-04-04T14:30:00Z - coder - lane=doing - Starting implementation
- 2026-04-04T14:45:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-04T23:00:00Z - review-coordinator - lane=to_do - Verdict: Changes Required (1 FAIL) -- awaiting remediation
