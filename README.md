# Skills Repository

This repository contains a collection of **skills** compatible with the `npx skills add` ecosystem. Each skill is self‑contained and can be installed individually.

## Available Skills

- **architecture-audit** – Comprehensive architecture audit that combines ruthless analysis with solution‑focused improvement planning.
- **itemized-functions** – Generate exhaustive integration functions with comprehensive test suites for all 3rd-party APIs and external services. Automatically creates function wrappers, individual test files, integrated test runners, and a detailed report of API behavior, response signatures, latency, and failure modes.

## Installation

Add the entire repository (all skills):

```bash
npx skills add harshitsinghbhandari/domain-expansion
```

## Structure

```
skills/
  architecture-audit/
    SKILL.md
    plugin.json
  itemized-functions/
    SKILL.md
    plugin.json

README.md
skills.json
```

Each skill folder contains a `SKILL.md` defining the skill and a `plugin.json` describing the entry point.
