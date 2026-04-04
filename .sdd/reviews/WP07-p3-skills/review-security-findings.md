---
skill: review-security
wp: WP07-p3-skills
spec: .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
reviewed_at: 2026-04-05T12:00:00Z
status: completed
finding_counts:
  pass: 3
  warn: 3
  fail: 0
  na: 13
files_reviewed:
  - .github/skills/review-performance/SKILL.md
  - .github/skills/review-docs/SKILL.md
  - .github/skills/review-deps/SKILL.md
  - .sdd/plans/WP07-p3-skills.md
  - .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
---

# review-security Findings for WP07-p3-skills

## Summary

Reviewed three P3 review skill files (review-performance, review-docs, review-deps) against the 14 OWASP Secure Coding Practices categories and the spec's security NFRs (Section 10.2). The deliverables are Markdown instruction files that guide an AI subagent through a review checklist — they are not executable application code. Consequently, 13 of 14 OWASP categories are not applicable. Security evaluation focuses on whether the skill instructions adequately propagate the spec's security constraints (NFR-004, NFR-005, NFR-006) to the executing agent.

Overall security posture: **Good with minor gaps.** All three skills correctly enforce the read-only constraint (FR-028). No secrets, credentials, or hardcoded sensitive data exist in the files. However, all three skills omit explicit constraints for NFR-004 (no code execution) and NFR-005 (no secret reproduction) that the P1 review-security skill includes. The review-deps skill references NFR-006 trusted sources but lacks the explicit prohibition against fetching arbitrary URLs from the codebase.

## Findings

### SEC-001 [PASS]
- **Checklist item**: Data Protection - Read-only constraint (FR-028)
- **Requirement**: FR-028, OWASP SCP Category 8
- **File**: .github/skills/review-performance/SKILL.md#L12, .github/skills/review-docs/SKILL.md#L14, .github/skills/review-deps/SKILL.md#L13
- **Description**: All three skills explicitly state a read-only constraint prohibiting modification of source code, the WP file, or the spec file, with output limited to the specified findings path. This matches the common skill contract (FR-028) and the P1 skill pattern.

### SEC-002 [WARN]
- **Checklist item**: General Coding Practices - No code execution (NFR-004)
- **Requirement**: NFR-004, OWASP SCP Category 14
- **File**: .github/skills/review-performance/SKILL.md#L10-L13, .github/skills/review-docs/SKILL.md#L12-L15, .github/skills/review-deps/SKILL.md#L11-L14
- **Description**: All three skills omit an explicit "Do NOT execute any code from the codebase" constraint. The P1 review-security skill includes this as a top-level constraint: "Do NOT execute any code from the codebase - review is static analysis only (NFR-004)." The P3 skills state only the FR-028 read-only constraint (no modification), which does not cover execution. An agent following review-performance could, in theory, decide to run a benchmark or profile code to check performance — the instructions don't prohibit it.
- **Expected**: Each skill's constraint block should include an explicit NFR-004 guard, e.g.: "Do NOT execute any code from the codebase - review is static analysis only (NFR-004)."
- **Evidence**:
  ```markdown
  # review-performance SKILL.md constraints:
  **Constraint**: Do NOT modify any source code, the WP file, or the spec file. Only write to the specified output path (FR-028).

  # review-security SKILL.md constraints (P1 reference):
  **Constraints**:
  - Do NOT modify any source code, the WP file, or the spec file.
  - Do NOT execute any code from the codebase - review is static analysis only (NFR-004).
  - Do NOT reproduce actual secret values (API keys, tokens, passwords) in findings - cite file and line only (NFR-005).
  ```

### SEC-003 [WARN]
- **Checklist item**: Data Protection - No secret reproduction in findings (NFR-005)
- **Requirement**: NFR-005, OWASP SCP Category 8
- **File**: .github/skills/review-performance/SKILL.md#L10-L13, .github/skills/review-docs/SKILL.md#L12-L15, .github/skills/review-deps/SKILL.md#L11-L14
- **Description**: All three skills omit an explicit "Do NOT reproduce actual secret values in findings" constraint. The P1 review-security skill includes this. While review-performance is unlikely to encounter secrets, review-docs compares documentation against code (including configuration guides that may reference env vars or credentials), and review-deps examines dependency manifests that could contain registry tokens. Without the explicit prohibition, an agent could include secret values in evidence snippets.
- **Expected**: Each skill's constraint block should include an NFR-005 guard, e.g.: "Do NOT reproduce actual secret values (API keys, tokens, passwords) in findings - cite file and line only (NFR-005)."
- **Evidence**:
  ```markdown
  # review-docs SKILL.md constraints:
  **Constraint**: Do NOT modify any documentation files, source code, the WP file, or the spec file. Only write to the specified output path (FR-028).

  # Missing: NFR-005 constraint present in P1 review-security skill
  ```

### SEC-004 [WARN]
- **Checklist item**: Data Protection - Trusted URLs for web research (NFR-006)
- **Requirement**: NFR-006, OWASP SCP Category 8
- **File**: .github/skills/review-deps/SKILL.md#L37-L46
- **Description**: The review-deps skill lists trusted sources for CVE lookup under Category 1 with an "(NFR-006)" annotation, but does not include the explicit prohibition "Do NOT fetch arbitrary URLs from the codebase" that the P1 review-security skill includes as a top-level statement. The trusted source list is scoped to "CVE lookup" only — if the agent needs web research for other categories (e.g., checking if a package is abandoned, verifying license info), it has no guidance on URL restrictions for those lookups. The risk is that an agent examining dependency manifests could follow URLs found in source files (e.g., a repository URL in package.json) without restriction.
- **Expected**: Add a top-level constraint: "Do NOT fetch arbitrary URLs from the codebase (NFR-006). Only use trusted sources listed below." Move the trusted sources list to a top-level section (not under Category 1 only) and expand it to cover all categories that may use web research.
- **Evidence**:
  ```markdown
  # review-deps SKILL.md - trusted sources scoped to Category 1 only:
  **Trusted sources for CVE lookup** (NFR-006):
  - nvd.nist.gov (National Vulnerability Database)
  - github.com/advisories (GitHub Advisory Database)
  - npmjs.com (npm audit advisories)
  - pypi.org (Python package index)
  - crates.io (Rust crate registry)

  # review-security SKILL.md - top-level prohibition (P1 reference):
  **Do NOT fetch arbitrary URLs from the codebase** (NFR-006).
  ```

### SEC-005 [PASS]
- **Checklist item**: Data Protection - No secrets in source code
- **Requirement**: OWASP SCP Category 8
- **File**: .github/skills/review-performance/SKILL.md, .github/skills/review-docs/SKILL.md, .github/skills/review-deps/SKILL.md
- **Description**: All three skill files contain no hardcoded secrets, API keys, tokens, passwords, or credentials. The files are purely instructional Markdown with checklist items and template examples.

### SEC-006 [PASS]
- **Checklist item**: Error Handling - Graceful degradation for web research (review-deps)
- **Requirement**: OWASP SCP Category 7, FR-048
- **File**: .github/skills/review-deps/SKILL.md#L45-L46
- **Description**: The review-deps skill includes fallback guidance for web research failures: "If web research fails or is unavailable, record WARN: 'Unable to verify CVEs via external source. Manual audit recommended.'" This prevents the agent from silently skipping CVE checks without documenting the gap, and avoids hanging on failed requests.

### SEC-007 [N/A]
- **Checklist item**: Input Validation - All items
- **Justification**: Deliverables are Markdown instruction files, not application code that processes user input. No server-side validation, allow-lists, or input canonicalization is applicable.

### SEC-008 [N/A]
- **Checklist item**: Output Encoding - All items
- **Justification**: Deliverables are Markdown instruction files. No output rendering, HTML/JS/CSS context encoding, or SQL/XML/LDAP escaping is applicable.

### SEC-009 [N/A]
- **Checklist item**: Authentication and Password Management - All items
- **Justification**: Deliverables do not implement authentication, credential storage, or login flows. The skills review code but do not handle credentials themselves.

### SEC-010 [N/A]
- **Checklist item**: Session Management - All items
- **Justification**: No session creation, management, or cookie handling in instruction files.

### SEC-011 [N/A]
- **Checklist item**: Access Control - All items
- **Justification**: No authorization logic, RBAC/ABAC enforcement, or access control decisions in instruction files.

### SEC-012 [N/A]
- **Checklist item**: Cryptographic Practices - All items
- **Justification**: No cryptographic operations, key management, or random number generation in instruction files.

### SEC-013 [N/A]
- **Checklist item**: Error Handling and Logging (application-level) - All items
- **Justification**: Instruction files define advisory error handling guidance for the executing agent (e.g., web research fallback) but do not implement application-level error handling, logging infrastructure, or security event capture.

### SEC-014 [N/A]
- **Checklist item**: Communication Security - All items
- **Justification**: Instruction files do not establish network connections, configure TLS, or transmit data. Web research is performed by the agent framework, not by the skill file itself.

### SEC-015 [N/A]
- **Checklist item**: System Configuration - All items
- **Justification**: No deployable application components, HTTP servers, or configurable system settings in instruction files.

### SEC-016 [N/A]
- **Checklist item**: Database Security - All items
- **Justification**: No database access, queries, or connection management in instruction files.

### SEC-017 [N/A]
- **Checklist item**: File Management - All items
- **Justification**: Instruction files specify an output path for findings but do not implement file upload, dynamic includes, or redirect logic. File operations are handled by the agent framework.

### SEC-018 [N/A]
- **Checklist item**: Memory Management - All items
- **Justification**: Markdown files — no memory allocation, buffer management, or resource cleanup applicable.

### SEC-019 [N/A]
- **Checklist item**: General Coding Practices - All items (as application code)
- **Justification**: No executable code in deliverables. No OS command construction, dynamic code execution (eval/exec), or shared resource locking. NFR-004 compliance (the instruction-level prohibition against code execution by the agent) is evaluated separately in SEC-002.
