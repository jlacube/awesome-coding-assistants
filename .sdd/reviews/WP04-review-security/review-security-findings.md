---
skill: review-security
wp: WP04
spec: .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
reviewed_at: 2026-04-04T12:00:00Z
status: completed
finding_counts:
  pass: 7
  warn: 4
  fail: 0
  na: 10
files_reviewed:
  - .github/skills/review-security/SKILL.md
---

# review-security Findings for WP04

## Summary

Reviewed the `review-security` skill implementation (`.github/skills/review-security/SKILL.md`, 184 lines). The implementation is a Markdown instruction file that defines the OWASP-based security review checklist for use by a coordinator subagent. Since this is not executable application code, the majority of OWASP Secure Coding Practices categories are not applicable. Security evaluation focused on the file's handling of sensitive data constraints (NFR-005), network access restrictions (NFR-006), and code execution prevention (NFR-004). All three spec security NFRs are properly enforced. Four WARN findings relate to OWASP checklist items specified in FR-034 that are absent from the implementation, creating coverage gaps in future security reviews. No exploitable vulnerabilities or FAIL-level issues were found.

## Findings

### SEC-001 [N/A]
- **Checklist item**: Input Validation - Server-side validation
- **Justification**: Markdown instruction file; no user inputs are received or processed at runtime. All 6 input validation checklist items are N/A.

### SEC-002 [N/A]
- **Checklist item**: Output Encoding - Server-side encoding
- **Justification**: Markdown instruction file; no output is rendered to clients. All 3 output encoding checklist items are N/A.

### SEC-003 [N/A]
- **Checklist item**: Authentication and Password Management - Auth required for non-public resources
- **Justification**: No authentication mechanisms are implemented. The file contains instructions for reviewing authentication in target code but does not implement auth itself. All 6 authentication checklist items are N/A.

### SEC-004 [N/A]
- **Checklist item**: Session Management - Server-side session creation
- **Justification**: No sessions are created or managed. All 6 session management checklist items are N/A.

### SEC-005 [N/A]
- **Checklist item**: Access Control - Centralized authorization
- **Justification**: No access control is implemented. The file is read by the coordinator subagent without access restrictions. All 5 access control checklist items are N/A.

### SEC-006 [N/A]
- **Checklist item**: Cryptographic Practices - Approved algorithms only
- **Justification**: No cryptographic operations are performed. All 3 cryptographic practices checklist items are N/A.

### SEC-007 [PASS]
- **Checklist item**: Error Handling and Logging - Generic error messages, no sensitive data in errors
- **Requirement**: FR-036, OWASP SCP 2.7
- **File**: .github/skills/review-security/SKILL.md#L156-L158
- **Description**: Web research failure handling properly instructs the subagent to record a WARN and continue with checklist-based review, rather than exposing error details or halting. No sensitive data is included in error guidance.

### SEC-008 [PASS]
- **Checklist item**: Data Protection - No secrets in source code
- **Requirement**: NFR-005, OWASP SCP 2.8
- **File**: .github/skills/review-security/SKILL.md#L20-L21
- **Description**: The file contains no hardcoded secrets, API keys, or credentials. The output format examples use placeholder values only. The "Critical rule" section (line 176-177) explicitly prohibits reproducing actual secret values in findings, with a safe example pattern ("API key found as string literal assigned to API_KEY variable").

### SEC-009 [PASS]
- **Checklist item**: Communication Security - No fallback to insecure connections
- **Requirement**: NFR-006, OWASP SCP 2.9
- **File**: .github/skills/review-security/SKILL.md#L147-L158
- **Description**: Web research is restricted to a defined list of trusted domains (owasp.org, nvd.nist.gov, framework-specific docs). The file explicitly prohibits fetching arbitrary URLs from the codebase, preventing SSRF-like risks in the subagent's network access.

### SEC-010 [N/A]
- **Checklist item**: System Configuration - Latest approved versions, security headers
- **Justification**: No system components, servers, or HTTP responses are configured. All 4 system configuration checklist items are N/A.

### SEC-011 [N/A]
- **Checklist item**: Database Security - Parameterized queries
- **Justification**: No database access in this WP. The file contains instructions for reviewing database security in target code but does not access any database itself. All 4 database security checklist items are N/A.

### SEC-012 [PASS]
- **Checklist item**: File Management - No user-supplied data in file paths
- **Requirement**: FR-028, OWASP SCP 2.12
- **File**: .github/skills/review-security/SKILL.md#L180-L184
- **Description**: The output file path is provided by the coordinator via the subagent prompt, not constructed from user input. The skill writes only to the specified output path. The read-only constraint (FR-028) prevents modification of source code, WP files, or spec files.

### SEC-013 [N/A]
- **Checklist item**: Memory Management - Buffer checks
- **Justification**: Markdown instruction file; no memory allocation or buffer operations. All 3 memory management checklist items are N/A.

### SEC-014 [PASS]
- **Checklist item**: General Coding Practices - No dynamic code execution
- **Requirement**: NFR-004, OWASP SCP 2.14
- **File**: .github/skills/review-security/SKILL.md#L19
- **Description**: The file explicitly instructs: "Do NOT execute any code from the codebase - review is static analysis only (NFR-004)." This prevents the subagent from running discovered code, eliminating code injection and arbitrary execution risks.

### SEC-015 [PASS]
- **Checklist item**: Spec Security Cross-Reference - NFR-004 (no code execution)
- **Requirement**: NFR-004
- **File**: .github/skills/review-security/SKILL.md#L19
- **Description**: NFR-004 is explicitly cited in the Constraints section. The subagent is instructed that "review is static analysis only (reading and evaluating code, not running it)."

### SEC-016 [PASS]
- **Checklist item**: Spec Security Cross-Reference - NFR-005 (no secret reproduction)
- **Requirement**: NFR-005
- **File**: .github/skills/review-security/SKILL.md#L20-L21
- **Description**: NFR-005 is explicitly cited in the Constraints section and reinforced in the Severity Rules "Critical rule" section (line 176-177). The output format example demonstrates the safe pattern for reporting discovered secrets without reproducing values.

### SEC-017 [WARN]
- **Checklist item**: Spec Security Cross-Reference - FR-034 Category 6 coverage gap
- **Requirement**: FR-034
- **File**: .github/skills/review-security/SKILL.md#L78-L80
- **Expected**: Category 6 (Cryptographic Practices) should include a "FIPS 140-2 compliance where required" checklist item as specified in FR-034.
- **Description**: The spec's FR-034 lists "FIPS 140-2 compliance where required" as a key checklist item for Category 6 (Cryptographic Practices). This item is absent from the SKILL.md implementation. While conditional ("where required"), omitting it means the security skill will not check for FIPS compliance when reviewing codebases that require it.
- **Evidence**: FR-034 Category 6 specifies: "Approved algorithms only (no custom crypto), secure random number generation, proper key management, FIPS 140-2 compliance where required." The SKILL.md Category 6 includes only the first three items.

### SEC-018 [WARN]
- **Checklist item**: Spec Security Cross-Reference - FR-034 Category 11 coverage gap
- **Requirement**: FR-034
- **File**: .github/skills/review-security/SKILL.md#L107-L110
- **Expected**: Category 11 (Database Security) should include a "stored procedures for data abstraction" checklist item as specified in FR-034.
- **Description**: The spec's FR-034 lists "stored procedures for data abstraction" as a key checklist item for Category 11. This item is absent from the SKILL.md, creating a minor coverage gap.
- **Evidence**: FR-034 Category 11 specifies: "Parameterized queries (no SQL injection), least-privilege DB access, no hardcoded connection strings, stored procedures for data abstraction, default credentials changed." The SKILL.md includes four of the five items but omits stored procedures.

### SEC-019 [WARN]
- **Checklist item**: Spec Security Cross-Reference - FR-034 Category 12 coverage gap (open redirect)
- **Requirement**: FR-034
- **File**: .github/skills/review-security/SKILL.md#L113-L117
- **Expected**: Category 12 (File Management) should include a "no user data in redirects" checklist item as specified in FR-034.
- **Description**: The spec's FR-034 lists "no user data in redirects" as a key checklist item for Category 12 (File Management). This item is absent from the SKILL.md. Open redirect is a recognized vulnerability (CWE-601) where user-controlled data in redirect targets can enable phishing attacks. Omitting this check creates a coverage gap for open redirect detection.
- **Evidence**: FR-034 Category 12 specifies: "No user-supplied data in dynamic includes, auth before upload, file type validation by headers (not extension), upload directory execution disabled, no user data in redirects, no absolute paths to client." The SKILL.md includes five of the six items but omits the redirect check.

### SEC-020 [WARN]
- **Checklist item**: Spec Security Cross-Reference - FR-034 Category 13 coverage gap
- **Requirement**: FR-034
- **File**: .github/skills/review-security/SKILL.md#L120-L122
- **Expected**: Category 13 (Memory Management) should include a "null termination handling" checklist item as specified in FR-034.
- **Description**: The spec's FR-034 lists "null termination handling" as a key checklist item for Category 13. This item is absent from the SKILL.md. While often N/A for managed languages, the omission means the check won't be performed for C/C++ codebases.
- **Evidence**: FR-034 Category 13 specifies: "Buffer size checks, null termination handling, resource cleanup (not relying on GC), no known vulnerable functions." The SKILL.md includes three of the four items but omits null termination handling.

### SEC-021 [PASS]
- **Checklist item**: Spec Security Cross-Reference - NFR-006 (trusted URLs only)
- **Requirement**: NFR-006
- **File**: .github/skills/review-security/SKILL.md#L147-L158
- **Description**: NFR-006 is explicitly cited. The Web Research section defines a trusted domain list (owasp.org, nvd.nist.gov, framework-specific docs) and explicitly prohibits fetching arbitrary URLs from the codebase.
