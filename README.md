# Skills Repository

This repository contains a collection of **skills** compatible with the `npx skills add` ecosystem. Each skill is self‑contained and can be installed individually.

## Available Skills

- **architecture-audit** – Comprehensive architecture audit that combines ruthless analysis with solution‑focused improvement planning.

## Installation

Add the entire repository (all skills):

```bash
npx skills add <repo-url>
```

Add a specific skill:

```bash
npx skills add <repo-url>@architecture-audit
```

Replace `<repo-url>` with the URL of this repository (e.g., `https://github.com/yourname/skills-repo`).

## Structure

```
skills/
  architecture-audit/
    SKILL.md
    plugin.json

README.md
skills.json
```

Each skill folder contains a `SKILL.md` defining the skill and a `plugin.json` describing the entry point.
