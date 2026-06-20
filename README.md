# Skills Repository

This repository contains a collection of **skills** compatible with the `npx skills add` ecosystem. Each skill is self‑contained and can be installed individually.

## Available Skills

- **architecture-audit** – Comprehensive architecture audit that combines ruthless analysis with solution‑focused improvement planning.
- **code-quality-audit** – Strategic code quality review that combines Sukuna-style analysis with Gojo-style refactoring roadmaps.
- **code-refactor-executor** – Executes a multi-stage refactoring plan based on existing audit and improvements documentation.
- **test-coverage-audit** – Audit of test coverage and generation of missing test cases for your codebase.
- **itemized-functions** – Generate exhaustive integration functions with comprehensive test suites for all 3rd-party APIs and external services.
- **update-changelog** – Add, organize, and manage changelog entries in CHANGELOG.md following the Keep a Changelog format.
- **llm-council** – Convene an LLM Council of 5 advisors with different thinking styles for high-stakes decisions, with peer review and chairman synthesis.
- **llm-forge** – Run existing work through 5 specialist craftspeople (Surgeon, Architect, Sharpener, Humanizer, Stress-Tester) who each produce an improved version, with peer review and Master Forger synthesis.
- **resume-critic** – Critique resumes through 5 specialized reviewers, but only against a specific job description or concrete job title, with anonymous peer review and a final interview-readiness verdict.
- **resume-rewrite** – Rewrite a resume for a target job using only the candidate’s existing material, and only after a matching `resume-critic` review.
- **resume-upgrade** – Propose strategic resume upgrades like projects, skills, and proof-building moves for a target job, and only after a matching `resume-critic` review.
- **pr-review** – Comprehensive PR review focusing on code quality, test coverage, security, backward compatibility, and what CI cannot check.
- **pr-learning** – Extract actionable learnings from merged PRs by comparing initial submission vs final merged state.
- **edge-case-hunter** – Exhaustive edge-case review that hunts boundary conditions, missing guards, and unhandled failure modes with risk-scored results.
- **boundary-bug-hunter** – Aggressive user-flow / boundary-bug analysis on a diff or branch. Auto-detects entry points, traces flows through changed code, finds every seam (cross-module calls, serialization, file I/O, shared state, schema versioning, network/IPC), and refuses to mark the work complete until each unverified boundary has a real round-trip test or an explicit out-of-scope record.
- **spec-and-ship** – Interview-driven specification and execution workflow for non-trivial features, systems, refactors, integrations, or migrations built out through Agent Orchestrator (AO) workers. Interrogates intent and requirements before any code, then ships the spec into parallel execution.

## Installation

Add the entire repository (all skills):

```bash
npx skills add harshitsinghbhandari/domain-expansion
```

Alternatively, add a specific skill (e.g., `code-quality-audit`):

```bash
npx skills add harshitsinghbhandari/domain-expansion --skill code-quality-audit
```

## Structure

```
skills/
  architecture-audit/
    SKILL.md
    plugin.json
  code-quality-audit/
    SKILL.md
    plugin.json
  code-refactor-executor/
    SKILL.md
    plugin.json
  test-coverage-audit/
    SKILL.md
    plugin.json
  itemized-functions/
    SKILL.md
    plugin.json
  update-changelog/
    SKILL.md
    plugin.json
  llm-council/
    SKILL.md
    plugin.json
  llm-forge/
    SKILL.md
    plugin.json
  resume-critic/
    SKILL.md
    plugin.json
  resume-rewrite/
    SKILL.md
    plugin.json
  resume-upgrade/
    SKILL.md
    plugin.json
  pr-review/
    SKILL.md
    plugin.json
    bc-guidelines.md
    review-checklist.md
  pr-learning/
    SKILL.md
    plugin.json
  edge-case-hunter/
    SKILL.md
    plugin.json
  boundary-bug-hunter/
    SKILL.md
    plugin.json
    references/
      entry-point-detection.md
      test-frameworks.md
      boundary-types.md
      deflection-refusals.md
      audit-record-format.md
  spec-and-ship/
    SKILL.md
    plugin.json
    IDEATION.md
    EXECUTION.md
    TEMPLATE.md

README.md
CHANGELOG.md
skills.json
```

Each skill folder contains a `SKILL.md` defining the skill and a `plugin.json` describing the entry point.
