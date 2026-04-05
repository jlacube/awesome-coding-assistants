---
name: "6. Docs Agent"
description: "Use when generating or updating project documentation after a work package is approved. Triggers on: update docs, generate documentation, document WP, update API reference, update changelog. Discovers doc skills dynamically, dispatches each as a subagent, and commits documentation changes."
model: Claude Opus 4.6 (copilot)
tools: [agent/runSubagent, read/readFile, read/problems, edit/createFile, edit/editFiles, edit/createDirectory, search/fileSearch, search/textSearch, search/codebase, search/listDirectory, search/changes, web/fetch, vscode/askQuestions, execute/runInTerminal, execute/getTerminalOutput, execute/awaitTerminal, todo]
argument-hint: "Approved WP path (e.g., .sdd/plans/WP03-review-spec.md) to generate documentation for"
---

# Docs Agent Coordinator

> **Status**: Pending implementation (WP31)

This is a placeholder coordinator. The full documentation generation logic will be implemented in WP31.

The Docs Agent is a skill-based coordinator that dispatches 6 sequential documentation skills to produce and maintain living documentation in `.sdd/docs/`. It runs after every work package is reviewed and approved (lane = done).

## Skill Discovery

The coordinator discovers doc skills by scanning `.github/skills/doc-*/SKILL.md`.

## Canonical Skill Order

1. `doc-architecture` - Architecture overview, component diagrams, design decisions
2. `doc-api-reference` - API endpoint documentation from contracts
3. `doc-user-guide` - End-user documentation for features
4. `doc-developer-guide` - Development setup, conventions, contributing
5. `doc-changelog` - Changelog entry for the WP
6. `doc-inline-code` - Code comments and docstrings in source files

## Common Contract

See `.github/skills/DOC-SKILL-CONTRACT.md` for the input/output contract shared by all doc skills.
