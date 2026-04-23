---
name: pr-review
description: Comprehensive PR review focusing on code quality, test coverage, security, backward compatibility, and what CI cannot check. Use when reviewing PRs, when asked to review code changes, or when the user mentions "review PR", "code review", or "check this PR".
---

# PR Review Skill

Review pull requests focusing on what CI cannot check: code quality, design decisions, test coverage adequacy, security vulnerabilities, and backward compatibility.

## Usage Modes

### No Argument

If the user invokes `/pr-review` with no arguments, **do not perform a review**. Instead, ask the user what they would like to review:

> What would you like me to review?
> - A PR number or URL (e.g., `/pr-review 12345`)
> - A local branch (e.g., `/pr-review branch`)

### Local CLI Mode

The user provides a PR number or URL:

```
/pr-review 12345
/pr-review https://github.com/owner/repo/pull/12345
```

For a detailed review with line-by-line specific comments:

```
/pr-review 12345 detailed
/pr-review https://github.com/owner/repo/pull/12345 detailed
```

Use `gh` CLI to fetch PR data:

```bash
# Get PR details
gh pr view <PR_NUMBER> --json title,body,author,baseRefName,headRefName,files,additions,deletions,commits

# Get the diff
gh pr diff <PR_NUMBER>

# Get PR comments and reviews
gh pr view <PR_NUMBER> --json comments,reviews

# Get line-level inline review comments (not included in comments,reviews)
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments
```

### Local Branch Mode

Review changes in the current branch that are not in `main`:

```
/pr-review branch
/pr-review branch detailed
```

Use git commands to get branch changes:

```bash
# Get current branch name
git branch --show-current

# Get list of changed files compared to main
git diff --name-only main...HEAD

# Get full diff compared to main
git diff main...HEAD

# Get commit log for the branch
git log main..HEAD --oneline

# Get diff stats (files changed, insertions, deletions)
git diff --stat main...HEAD
```

For local branch reviews:
- The "Summary" should describe what the branch changes accomplish based on commit messages and diff
- Use the current branch name in the review header instead of a PR number
- All other review criteria apply the same as PR reviews

### GitHub Actions Mode

When invoked via `@claude /pr-review` on a GitHub PR, the action pre-fetches PR
metadata and injects it into the prompt. Detect this mode by the presence of
`<formatted_context>`, `<pr_or_issue_body>`, and `<comments>` tags in the prompt.

The prompt already contains:
- PR metadata (title, author, branch names, additions/deletions, file count)
- PR body/description
- All comments and review comments (with file/line references)
- List of changed files with paths and change types

Use git commands to get the diff and commit history. The base branch name is in the
prompt context (look for `PR Branch: <head> -> <base>` or the `baseBranch` field).

```bash
# Get the full diff against the base branch
git diff origin/<baseBranch>...HEAD

# Get diff stats
git diff --stat origin/<baseBranch>...HEAD

# Get commit history for this PR
git log origin/<baseBranch>..HEAD --oneline

# If the base branch ref is not available, fetch it first
git fetch origin <baseBranch> --depth=1
```

Do NOT use `gh` CLI commands in this mode -- only git commands are available.
All PR metadata, comments, and reviews are already in the prompt context;
only the diff and commit log need to be fetched via git.

## Review Philosophy

A single line of code can have deep cross-cutting implications: a missing null check causes silent failures, a missing validation opens security holes, a manual implementation instead of using framework utilities creates maintenance burden. **Treat every line as potentially load-bearing.**

1. **Investigate, don't guess** — When uncertain whether a checklist item applies, spawn a sub-agent to read the relevant code. A reviewer who guesses wrong provides negative value.
2. **Review the design, not just the implementation** — A PR can have perfectly correct implementation of a bad design. Question side-channel communication, on/off private flags, and demand concrete interface documentation for new contracts between components.
3. **Focus on what CI cannot check** — Don't comment on formatting, linting, type errors, or CI failures. Focus on design quality, interface correctness, thread safety, BC implications, test adequacy, and pattern adherence.
4. **Everything is a must-fix** — There are no "nits." If it's worth mentioning, it's worth fixing. Every inconsistency degrades the codebase over time.
5. **Be specific and actionable** — Reference file paths and line numbers. Name the function/class/file the author should use.
6. **Match the immediate context** — Read how similar features are already implemented in the same file. Pattern mismatches within a file are always wrong.
7. **Assume competence** — The author knows the codebase; explain only non-obvious context.
8. **No repetition** — Each observation appears in exactly one section of the review output.
9. **If you noticed it, report it** — If something looks fragile, hacky, or wrong, include it in the review. Do not rationalize it away as "pragmatic," "standard practice," or "not worth mentioning." The author can decide it's acceptable; your job is to surface it.
10. **Default to skepticism** — CI passing is necessary but not sufficient. Approve only with full confidence. "Request Changes" is the safe default. Fragile pairings, version-pinned overrides, lockfile corruption, and unsupported dependency combinations all pass CI today and break tomorrow.
11. **Output goes under the user's name** — Your review is posted as their review. Every approval, missed issue, and recommendation reflects on them personally. Review as if your professional reputation depends on it — because theirs does.

### Using Sub-Agents

The review checklist is large. You cannot hold the full context of every infrastructure system in your head. **Spawn sub-agents** to investigate whether checklist items apply: read surrounding code, infrastructure the PR should be using, or tests that should exist. Spawn them in parallel for independent areas. A typical medium PR should spawn 3-8 sub-agents.

## Review Workflow

### Step 1: Understand Context

Before reviewing, build understanding of what the PR touches and why:
1. Identify the purpose of the change from title/description/issue
2. Group changes by type (new code, tests, config, docs)
3. Note the scope of changes (files affected, lines changed)
4. Spawn sub-agents to read the unchanged code surrounding each significantly changed file to understand existing patterns and infrastructure

### Step 2: Deep Review

Go through **every changed line** in the diff and evaluate it against the review checklist in [review-checklist.md](review-checklist.md).

### Step 2b: Data Flow Audit

Before evaluating individual lines, trace the data that crosses persistence boundaries. This catches bugs invisible to static code analysis — the kind where code looks correct in isolation but data is lost or corrupted as it flows through the system.

**Identify data shapes** that cross a persistence boundary: written to disk, database, config file, API response, network protocol, or external tool state.

**For each data shape, trace the full round-trip:**

1. **Who creates it?** What schema validates it on write?
2. **Who stores it?** What format (JSON, YAML, key=value, binary)?
3. **Who reads it back?** What schema validates it on read?
4. **Who transforms it?** Are there intermediate formats or sanitization steps?
5. **Who writes it again?** Does the re-write preserve all fields?

**Specific checks for each data shape:**

- **Type vs runtime validator alignment** — If the project uses both compile-time types (TypeScript `type`/`interface`, Python type hints) AND a runtime validator (Zod, Joi, Pydantic, JSON Schema), diff them. Every value in the compile-time type MUST exist in the runtime validator schema. Missing values are silently stripped on parse. Search for `z.object`, `z.enum`, `z.union`, `BaseModel`, `Schema` and compare against corresponding TypeScript types or Python type hints.

- **Dual-path consistency** — If the same conceptual data has two code paths (two readers, two writers, a reader and a writer), verify they agree on schema, field priority, and transformation rules. Example: path A does `storedValue ?? derivedValue` but path B does `derivedValue ?? storedValue` — one is wrong. Find all readers of a value and diff their derivation logic.

- **Sanitizer strips data needed later** — If a sanitizer/normalizer processes raw input before storage, verify it doesn't strip fields that downstream code relies on. "Preserved until X" comments are a red flag — trace whether preservation actually works end-to-end through the write→parse→re-write cycle.

- **Counter/flag writer-reader alignment** — For every `if (counter > 0)` or `if (flag)`, trace who increments/sets it. Are there OTHER code paths that should also increment it but don't? Writer-reader mismatches are invisible when you look at either side alone.

**Check external tool contracts:** If the PR moves, renames, or manages resources tracked by external tools (git worktrees, tmux sessions, Docker containers, k8s resources), verify the tool's internal state is updated — not just the filesystem. Examples: `git worktree repair` after moving a worktree, `tmux rename-session` after renaming a session.

**Check partial operation crash safety:** If a function performs N sequential filesystem/network operations, verify what happens if it crashes after step K. Is the state recoverable? Can the operation be safely re-run (idempotent)? Are temp files cleaned up on failure?

Spawn sub-agents for each data shape to trace in parallel.

### Step 3: Review Existing PR Discussion

Before formulating your review, read what others have already said on the PR:

1. **In CLI mode:** Fetch inline review comments via `gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments` and PR-level reviews via `gh pr view <PR_NUMBER> --json comments,reviews`
2. **In GitHub Actions mode:** Read the `<comments>` section already in the prompt context
3. **Acknowledge valid concerns** — If another reviewer (human or bot, including Copilot) raised a legitimate issue, incorporate or reference it. Do not silently ignore existing discussion.
4. **Disagree explicitly** — If you believe an existing concern is wrong, say so with reasoning. Silence is not disagreement.
5. **Don't duplicate** — Reference existing comments instead of repeating them.

### Step 4: Check Backward Compatibility

Evaluate BC implications per [bc-guidelines.md](bc-guidelines.md). For non-trivial BC questions, spawn a sub-agent to search for existing callers of the modified API.

### Step 5: Edge Case Hunt on High-Risk Files

If the PR touches high-risk code (state machines, parsers, dependency resolution, auth flows, financial calculations, concurrency, or configuration merging), invoke `/edge-case-hunter` on the changed files in those areas. Include the top findings in the Edge Cases section of your review.

Skip this step only if the PR is purely additive (new files, new tests) with no modifications to existing complex logic.

### Step 6: Formulate Review

Structure your review with actionable feedback organized by category. Every finding should be traceable to a specific line in the diff.

## Output Format

Structure your review as follows. **Omit sections where you have no findings** — don't write "No concerns" for every empty section. Only include sections with actual observations.

```markdown
## PR Review: #<number>
<!-- Or for local branch reviews: -->
## Branch Review: <branch-name> (vs main)

### Summary
Brief overall assessment of the changes (1-2 sentences).

### Code Quality
[Issues and suggestions]

### Architecture & Design
[Flag any checklist items from the Infrastructure section that apply.
Reference the specific patterns or utilities the PR should be using.]

### Testing
[Testing adequacy findings — missing edge cases, non-device-generic tests, etc.]

### API Design
[Flag new patterns, internal-access flags, or broader implications if any.]

### Security
[Issues if any]

### Thread Safety
[Threading concerns if any]

### Backward Compatibility
[BC concerns if any]

### Data Integrity
[Round-trip issues, schema mismatches (compile-time type vs runtime validator), dual-path inconsistencies, sanitizer stripping, counter/flag writer-reader mismatches, external tool contract breaks, crash safety gaps. Only present when the PR touches persistence, serialization, config loading, or external tool state.]

### Edge Cases
[High-risk edge cases found by targeted analysis of critical changed files.
Only present when the PR touches state machines, parsers, dependency resolution,
auth flows, financial calculations, or similarly complex logic.]

### Performance
[Performance concerns if any]

### Recommendation
**Approve** / **Request Changes** / **Needs Discussion**

[Brief justification for recommendation]
```

### Specific Comments (Detailed Review Only)

**Only include this section if the user requests a "detailed" or "in depth" review.**

**Do not repeat observations already made in other sections.** This section is for additional file-specific feedback that doesn't fit into the categorized sections above.

When requested, add file-specific feedback with line references:

```markdown
### Specific Comments
- `src/module.py:42` - Consider extracting this logic into a named function for clarity
- `test/test_feature.py:100-105` - Missing test for error case when input is None
- `src/utils/helper.ts:78` - This allocation could be moved outside the loop
```

## Files to Reference

When reviewing, consult these project files for context — read them rather than relying on memory, as they change frequently:
- `CLAUDE.md` or `AGENTS.md` - Coding style philosophy and project-specific patterns
- `CONTRIBUTING.md` - PR requirements and review process
- Project-specific test utilities and patterns
- Configuration files that define project conventions
