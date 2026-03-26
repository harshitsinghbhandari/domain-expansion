---
name: update-changelog
description: Expert skill for maintaining a Keep a Changelog formatted CHANGELOG.md file. Use this skill whenever you add features, fix bugs, or release a new version. You MUST use this skill to record any changes that have a user-facing impact. It handles categorization (Added, Changed, Fixed, etc.), semantic versioning, and reverse-chronological ordering with surgical precision.
---

# Update Changelog Skill

## Overview

Maintain accurate, user-facing changelog entries as you build. This skill helps you document new features, fixes, enhancements, and breaking changes in a structured format that follows [Keep a Changelog](https://keepachangelog.com/).

The changelog is your project's narrative—it tells users what changed and why. AI agents should document their work just as humans do.

---

## When to Use This Skill

Use this skill whenever you:
- **Add a new feature** — enhancement that extends functionality
- **Fix a bug** — resolution of unexpected behavior
- **Deprecate something** — warn users about upcoming removal
- **Remove functionality** — breaking change that affects users
- **Improve performance** — optimization or security hardening
- **Release a version** — formalize changes under a version number

---

## The Changelog Structure

The CHANGELOG.md file has this structure:

```
# Changelog

## [Unreleased]
### Added
### Changed
### Deprecated
### Removed
### Fixed
### Security

## [Version] - [Date]
### Added
...
```

**Important sections:**
- `[Unreleased]` — changes that haven't been released yet (always at the top)
- Categories — Added, Changed, Deprecated, Removed, Fixed, Security (use all that apply)
- Entries — clear, user-facing descriptions (not implementation details)

---

## Workflow: Adding an Entry

### Step 1: Identify the Category

Decide which category your change belongs to:

| Category | When to Use | Example |
|----------|-----------|---------|
| **Added** | New feature or capability | "Add priority-based task queuing for agents" |
| **Changed** | Behavior or API modification | "Improve agent coordination algorithm" |
| **Deprecated** | Mark something for future removal | "Deprecate legacy executor API" |
| **Removed** | Delete functionality | "Remove deprecated tmux executor support" |
| **Fixed** | Bug resolution | "Fix agent crash when PR has no comments" |
| **Security** | Vulnerability or safety fix | "Fix privilege escalation in plugin system" |

### Step 2: Write a Clear Entry

Write entries from the **user's perspective**, not implementation details.

✅ **Good entries:**
- "Add ability to prioritize high-urgency tasks in the orchestrator"
- "Improve agent responsiveness during CI failures by 40%"
- "Fix agents hanging on repos with very large file trees"

❌ **Avoid:**
- "Refactor TaskQueue class"
- "Update dependencies"
- "Merge branch feature/xyz"

### Step 3: Add References (Optional)

If relevant, link to the PR, issue, or commit:

```markdown
- Added new plugin system for extensible agent behaviors ([PR #42](https://github.com/your-org/your-repo/pull/42))
- Fixed memory leak in event listener ([Issue #89](https://github.com/your-org/your-repo/issues/89))
```

### Step 4: Open CHANGELOG.md and Locate [Unreleased]

Find the `## [Unreleased]` section at the top of the file.

### Step 5: Add Your Entry Under the Correct Category

Add your entry as a bullet point under the relevant category in **[Unreleased]**:

```markdown
## [Unreleased]

### Added
- New feature description goes here
- Another feature

### Fixed
- Bug fix description goes here
```

### Step 6: Review and Commit

- Verify the entry is clear and in the right category
- Make sure it reads naturally alongside other entries
- Commit with a message like: `docs: update changelog for [feature name]`

---

## Example: Adding a Feature

**What you did:** Built a new plugin system for agents.

**The entry you add to [Unreleased]/Added:**
```markdown
- Add plugin system with extensible hooks for custom agent behaviors ([PR #42](https://github.com/your-org/your-repo/pull/42))
```

**In context (CHANGELOG.md):**
```markdown
## [Unreleased]

### Added
- Add plugin system with extensible hooks for custom agent behaviors ([PR #42](https://github.com/your-org/your-repo/pull/42))
- Improve agent task decomposition for better parallelization

### Fixed
- Fix agents hanging on repos with very large file trees
```

---

## Example: Fixing a Bug

**What you did:** Resolved agents crashing when processing empty PRs.

**The entry you add to [Unreleased]/Fixed:**
```markdown
- Fix agent crash when processing PRs with no body or comments
```

---

## Releasing a Version

When you're ready to formalize changes as a release:

### 1. Decide on a Version Number

Use [Semantic Versioning](https://semver.org/):
- **Major** (X.0.0) — breaking changes
- **Minor** (0.X.0) — new features (backward-compatible)
- **Patch** (0.0.X) — bug fixes (backward-compatible)

### 2. Create a New Version Section

Replace `[Unreleased]` with a new version heading:

```markdown
## [0.2.0] - 2025-03-26

### Added
- Add plugin system with extensible hooks for custom agent behaviors
- Improve agent task decomposition for better parallelization

### Fixed
- Fix agent crash when processing PRs with no body or comments

## [Unreleased]

### Added

### Changed

### Deprecated

### Removed

### Fixed

### Security
```

### 3. Update the Version Table (Optional)

If you have a version summary table, update it:

```markdown
| Version | Date | Highlights |
|---------|------|-----------|
| 0.2.0 | 2025-03-26 | Plugin system + better decomposition |
| 0.1.0 | 2025-03-01 | Foundation + initial setup |
```

### 4. Commit and Tag

```bash
git add CHANGELOG.md
git commit -m "release: v0.2.0"
git tag v0.2.0
git push origin main --tags
```

---

## Rules for Keeping Changelog Clean

1. **[Unreleased] always at the top** — new changes go here first
2. **Reverse chronological order** — newest versions first
3. **Clear, user-facing language** — explain what changed, not how
4. **One entry per line** — easy to scan and read
5. **Group related changes** — if features are interdependent, mention it
6. **Use backticks for code** — \`feature-name\`, \`PluginAPI\`, \`ConfigOption\`
7. **Link to PRs/issues** — when possible, for traceability
8. **Deprecation warnings** — announce removals 1-2 releases in advance

---

## Common Mistakes to Avoid

| Mistake | Wrong | Right |
|---------|-------|--------|
| Implementation details | "Refactor executor class" | "Improve executor performance by 30%" |
| Missing category | Entry floating in the file | Clearly under Added/Fixed/Changed/etc. |
| Too vague | "Various improvements" | "Improve agent coordination during peak load" |
| No user impact | "Update dependencies" | Only if users are affected (e.g., "Drop Python 3.8 support") |
| Typos | "Imprve agent responsivness" | Spell-check before committing |

---

## Hands-On: Your First Entry

### Practice Scenario

You just added a feature: agents can now run custom pre-flight checks before starting work.

**Your changelog entry:**

```markdown
### Added
- Allow agents to run custom pre-flight checks before task execution
```

**In context:**

```markdown
## [Unreleased]

### Added
- Allow agents to run custom pre-flight checks before task execution
```

✅ **Why this works:**
- User-facing (explains what agents can now do)
- Clear action verb (Allow, Add, Improve, Fix)
- Concise but complete
- In the right category

---

## Tips for Agents Adding Entries

When an AI agent is updating the changelog:

1. **Parse the current [Unreleased] section** — understand what's already there
2. **Determine the best category** — don't force-fit into the wrong section
3. **Write from the user's perspective** — "what does this change do for them?"
4. **Check for duplicates** — don't add the same feature twice
5. **Maintain alphabetical order** — optional but helpful for long lists
6. **Preserve formatting** — keep the markdown clean and readable
7. **Validate before committing** — ensure the file is still valid markdown

---

## File Paths

- **Changelog file:** `CHANGELOG.md` (root directory)
- **This skill:** `.claude/skills/update-changelog/SKILL.md` (or your skill directory)

---

## References

- [Keep a Changelog](https://keepachangelog.com/) — the standard this follows
- [Semantic Versioning](https://semver.org/) — how to number releases
- [Git Commit Message Conventions](https://www.conventionalcommits.org/) — structure your commits

---

## Quick Reference

```bash
# Workflow
1. Make changes to your codebase
2. Run this skill to add entries to CHANGELOG.md
3. Review the entries for clarity
4. Commit: git add CHANGELOG.md && git commit -m "docs: update changelog"
5. When releasing: create version section, commit, and tag

# Categories (in order)
- Added (new features)
- Changed (behavior changes)
- Deprecated (warn about removal)
- Removed (deleted features)
- Fixed (bug fixes)
- Security (vulnerabilities)

# Entry format
- Short, user-facing description
- Optional: link to PR/issue
- Start with action verb (Add, Improve, Fix, Allow, etc.)
```

---

**Last updated:** March 2025  
**Maintained with:** Agent Orchestrator by Composio (MIT License)