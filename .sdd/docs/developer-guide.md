# Developer Guide

## Project Structure

```
.github/
  agents/                           # VS Code Copilot Chat agent definitions
    review-coordinator.agent.md     # Review orchestration agent
    orchestrator.agent.md           # Pipeline orchestration agent
    coder.agent.md                  # Implementation coordinator (dispatches coding skills)
    docs-agent.agent.md             # Documentation generation agent
    spec-architect.agent.md         # Specification agent
    planner.agent.md                # Planning agent
  schemas/
    enums.yaml                      # Central enum registry -- single source of truth for pipeline enums and conventions (WP40)
    orchestrator-handoff.schema.yaml
    coder-to-reviewer.schema.yaml
    reviewer-to-coder.schema.yaml
    reviewer-to-spec.schema.yaml
    planner-to-coder.schema.yaml
    planner-to-spec.schema.yaml
    spec-to-planner.schema.yaml
    ideation-to-spec.schema.yaml
  skills/                           # VS Code Copilot Chat skill definitions
    code-env-setup/SKILL.md         # Environment setup coding skill
    code-implementation/SKILL.md    # Core implementation coding skill
    code-unit-tests/SKILL.md        # Unit test coding skill
    code-integration-tests/SKILL.md # Integration test coding skill
    code-debug/SKILL.md             # Debug coding skill
    CODER-SKILL-CONTRACT.md         # Common contract for all coding skills
    review-spec/SKILL.md            # Spec adherence review skill
    review-security/SKILL.md        # Security review skill
    review-quality/SKILL.md         # Code quality review skill
    review-tests/SKILL.md           # Test quality review skill
    review-architecture/SKILL.md    # Architecture review skill
    review-performance/SKILL.md     # Performance review skill
    review-docs/SKILL.md            # Documentation review skill
    review-deps/SKILL.md            # Dependency review skill
    semantic-commit/SKILL.md        # Commit grouping skill (pre-existing)

.sdd/
  ideas/                            # Ideation briefs
  specs/                            # Specifications
  plans/                            # Work package plans + README.md index
  reviews/                          # Review artifacts
    review-patterns.md              # Active + resolved review patterns
    <WP-id>/                        # Per-WP findings (one dir per reviewed WP)
  docs/                             # Project documentation (this directory)
```

## Adding a New Review Skill

The review system discovers skills dynamically. To add a new review dimension:

### 1. Create the skill directory

```
.github/skills/review-<name>/SKILL.md
```

The directory name MUST match the pattern `review-*`.

### 2. Write the SKILL.md file

The skill file must contain:

**YAML frontmatter**:
```yaml
---
name: "review-<name>"
description: "Description of what this skill reviews"
---
```

**Body sections**:
1. **Purpose statement** - What this skill reviews and why
2. **Checklist** - Organized by category, each item assessable as PASS/WARN/FAIL/N/A
3. **Severity guidance** - Which items are FAIL vs WARN
4. **Output format** - Structured findings format instruction

### 3. Define the findings format

Each skill uses a unique finding ID prefix (e.g., `SPEC-`, `SEC-`, `QUAL-`). Findings files use this structure:

**YAML frontmatter**:
```yaml
---
skill: review-<name>
wp: <WP-id>
spec: <spec-path>
reviewed_at: <ISO 8601 timestamp>
status: completed
finding_counts:
  pass: <n>
  warn: <n>
  fail: <n>
  na: <n>
files_reviewed:
  - <file-path-1>
  - <file-path-2>
---
```

**Finding entries**:
```markdown
### <PREFIX>-<NNN> [<SEVERITY>]
- **Checklist item**: <which checklist item>
- **Requirement**: <FR-XXX or N/A>
- **File**: <path>#L<start>-L<end>
- **Description**: <what was found>
- **Expected**: <what should be true>
- **Evidence**: <code snippet or observation>
```

### 4. No coordinator changes needed

The coordinator discovers skills by scanning `.github/skills/review-*/SKILL.md` at runtime. Your new skill will be dispatched automatically in the next review.

Skills not in the canonical order list are dispatched after all canonical skills, sorted alphabetically.

### Canonical dispatch order

1. review-spec
2. review-security
3. review-quality
4. review-tests
5. review-architecture
6. review-performance
7. review-docs
8. review-deps
9. (any additional skills, alphabetically)

## Skill Input Contract

Each skill subagent receives a prompt containing:

1. **skill_path** - Path to the SKILL.md file
2. **wp_id** - Work package identifier (e.g., `WP01`)
3. **spec_path** - Path to the specification file
4. **output_path** - Path to write the findings file
5. **previous_findings_path** (re-review only) - Path to previous findings

## Skill Constraints

- Skills are **read-only** -- they must NOT modify source code, WP files, spec files, or any file other than the output findings file
- Skills are **stateless** -- each invocation is independent
- Skills must be **under 300 lines** to fit in a subagent context window
- Skills must mark inapplicable checklist items as **N/A** with justification
- Finding IDs must be **unique within the skill** (prefixed by skill domain)

## Review Patterns

The coordinator maintains `.sdd/reviews/review-patterns.md` with patterns extracted from FAIL findings. The Coder agent reads this file before implementing each WP.

- Only FAIL findings generate patterns (not WARNs)
- Patterns are moved to Resolved when they stop recurring
- Pattern IDs (PAT-NNN) are never reused

## Conventions

- All timestamps use **ISO 8601** format (e.g., `2026-04-04T10:45:00Z`)
- All files use **plain ASCII** -- no em dashes, smart quotes, curly apostrophes, or non-breaking spaces
- Git commits use **explicit file listing** (`git add <file1> <file2>`) -- never `git add .`
- WP lifecycle: `planned` -> `doing` -> `for_review` -> `done` (or `to_do` for feedback, `blocked` for stalled cycles)
- **Enum values** are defined in `.github/schemas/enums.yaml` -- this is the single source of truth for `lane`, `spec_status`, `pipeline_stage`, and `review_status` values (WP40)
- Agent files referencing enum values include `<!-- Enum source: .github/schemas/enums.yaml -->` near those references (WP40)
- **Activity Log entries** use the canonical format: `<ISO-8601-timestamp> - <agent-name> - <action> - <details>` with ` - ` (space-hyphen-space) separators. Agent names are lowercase with hyphens (e.g., `coder`, `review-coordinator`, `docs-agent`) (WP40)
- The `spec_status` enum no longer includes "Final" -- valid values are Draft, Validated, Approved (WP40)
