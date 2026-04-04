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
- **pr-review** – Comprehensive PR review focusing on code quality, test coverage, security, backward compatibility, and what CI cannot check.

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
  pr-review/
    SKILL.md
    plugin.json
    bc-guidelines.md
    review-checklist.md

README.md
skills.json
```

Each skill folder contains a `SKILL.md` defining the skill and a `plugin.json` describing the entry point.
