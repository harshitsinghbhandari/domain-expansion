---
name: pr-learning
description: Extract actionable learnings from merged PRs by comparing initial submission vs final merged state. Analyzes review cycles to capture what changed and why. Use when the user says "learn from PR", "extract PR learnings", "what did we learn from this PR", or wants to build a knowledge base from merged PRs.
---

# PR Learning Extractor

Turn merged PRs into permanent, high-value learning artifacts by analyzing the full lifecycle: initial submission → review feedback → final merged state. Captures the delta between "what I thought was right" and "what was actually right."

## Why This Matters

The gap between a PR's first commit and its final merge represents concentrated learning:
- Review comments reveal blind spots in your thinking
- Code changes show patterns you didn't know existed
- Approval requirements expose team standards you weren't aware of

This skill extracts that knowledge before it's lost to scrollback.

**Your extraction goes under the user's name.** Learnings become part of their permanent knowledge base. Omitting a lesson or softening a finding wastes the review cycle. Extract the full, unvarnished lesson — the user can decide what to act on. If you noticed something worth learning, include it.

## When to Use

**Good triggers:**
- "Learn from PR #123"
- "Extract learnings from this PR"
- "What can we learn from PR 456?"
- "Add PR 789 to the knowledge base"
- "Run /pr-learning on the last 5 merged PRs"

**Not for:**
- Open PRs (no final state to compare against)
- PRs with no review comments (nothing to learn)
- Trivial PRs (dependency bumps, typo fixes)

## Usage Modes

### Single PR Mode

```
/pr-learning 12345
/pr-learning https://github.com/owner/repo/pull/12345
```

Analyzes one merged PR and appends learnings to both documents.

### Batch Mode

```
/pr-learning last 5
/pr-learning --since 2026-01-01
```

Processes multiple recent PRs. Useful for catching up on learnings.

### No Argument Mode

If invoked with no arguments, prompt the user:

> What PR would you like to extract learnings from?
> - A PR number or URL (e.g., `/pr-learning 12345`)
> - Recent PRs (e.g., `/pr-learning last 5`)

## Output Files

Every run updates two living documents:

### 1. Project-Specific Learnings

**Location:** `./learnings/pr-learnings.md` (in the repository)

Contains learnings specific to this codebase:
- Project conventions and patterns
- Module-specific gotchas
- Team preferences and standards
- Architecture decisions

### 2. General Learnings

**Location:** `~/learnings/general-learnings.md` (machine-wide)

Contains transferable knowledge:
- Language idioms and best practices
- Common review feedback patterns
- Security and performance principles
- Testing strategies

## Workflow

### Step 1: Validate PR State

```bash
# Get PR state
gh pr view <PR_NUMBER> --json state,merged,mergedAt,author

# Reject if not merged
```

If PR is not merged, respond:
> This PR hasn't been merged yet. Learning extraction works best on completed PRs where we can see the full review cycle. Would you like me to do a standard PR review instead?

### Step 2: Gather PR Lifecycle Data

Spawn 3 sub-agents in parallel to collect:

**Agent 1: Initial State**
```bash
# Get first commit(s) in PR
gh pr view <PR_NUMBER> --json commits

# Get the diff at first commit
git show <first_commit_sha> --stat
git show <first_commit_sha> --no-stat
```

**Agent 2: Review Comments & Feedback**
```bash
# Get all review comments
gh pr view <PR_NUMBER> --json reviews,comments

# Get review threads with context
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments
```

**Agent 3: Final State**
```bash
# Get final merged diff
gh pr diff <PR_NUMBER>

# Get commit messages showing evolution
gh pr view <PR_NUMBER> --json commits --jq '.commits[].messageHeadline'
```

### Step 3: Analyze the Delta

Compare initial submission vs final merge to identify:

1. **Code Changes After Review**
   - What code was added/removed/modified after review feedback?
   - Which review comments led to code changes?

2. **Pattern Corrections**
   - Where did reviewers suggest different patterns?
   - What conventions were enforced?

3. **Blind Spots Exposed**
   - What did the author miss that reviewers caught?
   - Security issues, edge cases, performance concerns?

4. **Knowledge Gaps**
   - What did the author not know about the codebase?
   - What framework features were suggested?

### Step 4: Classify Learnings

For each learning, determine:

| Field | Description |
|-------|-------------|
| **Category** | `pattern`, `convention`, `security`, `performance`, `testing`, `architecture`, `tooling` |
| **Scope** | `project` (specific to this repo) or `general` (transferable) |
| **Severity** | `critical` (would cause bugs), `important` (quality impact), `minor` (style/preference) |
| **Confidence** | `high` (explicit reviewer comment), `medium` (inferred from changes), `low` (speculative) |

### Step 5: Deduplicate Against Existing Learnings

Before adding a new learning, scan both documents for semantic duplicates:

```python
# Pseudo-logic for deduplication
for existing_learning in document:
    if semantic_similarity(new_learning, existing_learning) > 0.8:
        # Enhance existing instead of duplicating
        enhance_with_new_evidence(existing_learning, new_learning)
        return
# No match - add as new entry
append_learning(document, new_learning)
```

**Deduplication signals:**
- Same code pattern discussed
- Same file/module referenced
- Same reviewer feedback theme
- Same fix applied

If a near-duplicate exists, **enhance the existing entry** with:
- Additional evidence from the new PR
- Updated "frequency" count
- Cross-reference to the new PR

### Step 6: Write Learnings

**Project-specific document (`./learnings/pr-learnings.md`):**

```markdown
# PR Learnings - [Project Name]

> Auto-generated knowledge base from merged PR reviews.
> Last updated: [timestamp]

## Key Patterns (Auto-consolidated)

[Recurring themes extracted from 10+ PRs appear here]

---

## Recent Learnings

### [YYYY-MM-DD] PR #123: [PR Title]

**Author:** @username | **Reviewers:** @reviewer1, @reviewer2

#### Learning: [Concise title]

**Category:** `pattern` | **Severity:** `important`

**What happened:**
[Brief description of what the author originally submitted]

**What changed:**
[What reviewers requested and why]

**The lesson:**
[Actionable takeaway in imperative form]

**Evidence:**
- Review comment by @reviewer: "[quote]"
- Changed: `src/module.py:42-58`

**Related PRs:** #100, #89 (similar feedback)

---
```

**General learnings document (`~/learnings/general-learnings.md`):**

```markdown
# General Engineering Learnings

> Transferable knowledge extracted from PR reviews across all projects.
> Last updated: [timestamp]

## Patterns by Category

### Security
- [Learning entries...]

### Performance
- [Learning entries...]

### Testing
- [Learning entries...]

---

## Recent Additions

### [YYYY-MM-DD] From: owner/repo#123

**Learning:** [Title]

**Category:** `security` | **Confidence:** `high`

**Context:**
[What triggered this learning]

**The principle:**
[Generalized, transferable lesson]

**Example:**
```[language]
// Before (problematic)
[code]

// After (correct)
[code]
```

---
```

### Step 7: Smart Consolidation (Periodic)

Trigger consolidation when:
- Document exceeds 50KB
- 10+ new entries since last consolidation
- User explicitly requests: `/pr-learning consolidate`

**Consolidation process:**

1. **Extract recurring themes** from individual entries
2. **Create "Key Patterns" section** at document top
3. **Archive old entries** (move to bottom, collapse details)
4. **Update cross-references** between related learnings

### Step 8: Create Backup

Before any write operation:
```bash
# Backup existing files
cp ./learnings/pr-learnings.md "./learnings/pr-learnings_$(date +%Y%m%d_%H%M%S).md.bak"
```

Keep last 5 backups, delete older ones.

### Step 9: Report Summary

After processing, output:

```markdown
## PR Learning Extraction Complete

**PR:** #123 - [Title]
**Author:** @username
**Merged:** 2026-04-10

### Learnings Extracted: 3

| # | Learning | Category | Scope | Destination |
|---|----------|----------|-------|-------------|
| 1 | Use `pytest.raises` context manager | testing | general | ~/learnings/general-learnings.md |
| 2 | Always validate webhook signatures | security | general | ~/learnings/general-learnings.md |
| 3 | Service layer must not import from handlers | architecture | project | ./learnings/pr-learnings.md |

### Deduplication Actions: 1

- Enhanced existing learning "Prefer pytest fixtures over setUp" with new evidence from this PR

### Files Updated:
- `./learnings/pr-learnings.md` (2 new entries)
- `~/learnings/general-learnings.md` (2 new entries, 1 enhanced)
```

## Quality Filters

Not all PRs have extractable learnings. Skip or flag when:

| Condition | Action |
|-----------|--------|
| No review comments | Skip with message: "PR merged without review feedback - no learnings to extract" |
| All comments are nits | Flag: "Only minor style feedback - learnings may be low-value" |
| Author == only reviewer | Skip: "Self-merged PR - no external perspective" |
| PR is revert | Skip: "Revert PR - original PR may have learnings" |
| Trivial change | Skip: "Trivial change (dependency bump, typo fix)" |

## Example Session

**User:** `/pr-learning 456`

**Agent gathers data, then outputs:**

---

## PR Learning Extraction Complete

**PR:** #456 - Add rate limiting to public API endpoints
**Author:** @alice
**Merged:** 2026-04-08
**Reviewers:** @bob, @carol

### Analysis

**Initial submission:** Added rate limiting using a custom in-memory counter per endpoint.

**Review feedback:**
1. @bob: "We have a rate limiter middleware in `src/middleware/rate_limit.py` - please use that instead of rolling your own"
2. @carol: "In-memory counters won't work in our multi-pod deployment - need Redis-backed limiting"
3. @bob: "Missing tests for the 429 response case"

**Final changes:**
- Removed custom counter implementation
- Integrated existing `RateLimiter` middleware
- Added Redis backend configuration
- Added test for rate limit exceeded scenario

### Learnings Extracted: 3

#### 1. Check for existing infrastructure before implementing (Project)

**Category:** `architecture` | **Severity:** `important`

**The lesson:** Before implementing cross-cutting concerns (auth, rate limiting, caching, logging), search the codebase for existing solutions. This project has middleware in `src/middleware/` for common concerns.

**Files to check:** `src/middleware/`, `src/utils/`, `ARCHITECTURE.md`

---

#### 2. Stateless services need external state stores (General)

**Category:** `architecture` | **Severity:** `critical`

**The lesson:** In containerized/multi-instance deployments, in-memory state (counters, caches, sessions) won't work correctly. Use external stores (Redis, Memcached, database) for shared state.

**Pattern:**
```python
# Wrong: In-memory (fails in multi-pod)
request_counts = {}  # Lost on restart, not shared across pods

# Right: External store
redis_client.incr(f"rate_limit:{user_id}")
```

---

#### 3. Always test error responses (General)

**Category:** `testing` | **Severity:** `important`

**The lesson:** When adding features that can return error responses (rate limits, validation errors, auth failures), explicitly test those error paths including response codes and error message format.

---

### Files Updated:
- `./learnings/pr-learnings.md` (1 new entry)
- `~/learnings/general-learnings.md` (2 new entries)

---

## Important Rules

1. **Only process merged PRs** - Open PRs don't have final state
2. **Require review feedback** - No comments = no learnings
3. **Preserve evidence** - Always link to specific review comments
4. **Deduplicate aggressively** - Enhance existing over creating duplicates
5. **Classify scope correctly** - Project-specific stays in repo, general goes machine-wide
6. **Back up before writing** - Never lose existing learnings
7. **Keep learnings actionable** - "Do X" not "X is important"

## File Initialization

If learning files don't exist, create them with headers:

**`./learnings/pr-learnings.md`:**
```markdown
# PR Learnings - [Repository Name]

> Knowledge base auto-generated from merged PR reviews.
> Run `/pr-learning <PR>` to add learnings from any merged PR.

---

## Key Patterns

[Consolidated patterns will appear here after 10+ entries]

---

## Recent Learnings

[Individual PR learnings will be added below]
```

**`~/learnings/general-learnings.md`:**
```markdown
# General Engineering Learnings

> Transferable knowledge extracted from PR reviews across all projects.
> This file grows across all repositories you work in.

---

## Patterns by Category

### Security

### Performance

### Testing

### Architecture

### Tooling

---

## Recent Additions

[Learnings will be added below]
```

## Integration with PR Review

This skill complements `/pr-review`:
- **pr-review**: Reviews PRs *before* merge (prospective)
- **pr-learning**: Extracts lessons *after* merge (retrospective)

For maximum value, run `/pr-learning` on PRs that had significant review cycles.
