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

Your job is not to read the diff and check boxes. Your job is to **investigate** — trace every change through the system, experience it as a user would, and find the bugs that only surface when you look beyond the lines that changed.

A single line of code can have deep cross-cutting implications: a missing null check causes silent failures, a missing validation opens security holes, a manual implementation instead of using framework utilities creates maintenance burden. **Treat every line as potentially load-bearing.**

1. **Investigate, don't guess** — When uncertain whether something is a problem, spawn a sub-agent to read the relevant code. A reviewer who guesses wrong provides negative value.
2. **Review the design, not just the implementation** — A PR can have perfectly correct implementation of a bad design. Question hidden flags, side-channel communication, and undocumented contracts between components.
3. **Focus on what CI cannot check** — Don't comment on formatting, linting, type errors, or CI failures. Focus on design quality, interface correctness, thread safety, BC implications, test adequacy, and pattern adherence.
4. **Everything is a must-fix** — There are no "nits." If it's worth mentioning, it's worth fixing. Every inconsistency degrades the codebase over time.
5. **Be specific and actionable** — Reference file paths and line numbers. Name the function/class/file the author should use.
6. **Match the immediate context** — Read how similar features are already implemented in the same file. Pattern mismatches within a file are always wrong.
7. **Assume competence** — The author knows the codebase; explain only non-obvious context.
8. **No repetition** — Each observation appears in exactly one section of the review output.
9. **If you noticed it, report it** — If something looks fragile, hacky, or wrong, include it in the review. Do not rationalize it away as "pragmatic," "standard practice," or "not worth mentioning." The author can decide it's acceptable; your job is to surface it.
10. **Default to skepticism** — CI passing is necessary but not sufficient. Approve only with full confidence. "Request Changes" is the safe default.
11. **Output goes under the user's name** — Your review is posted as their review. Every approval, missed issue, and recommendation reflects on them personally. Review as if your professional reputation depends on it — because theirs does.

### Using Sub-Agents

The review checklist is large. You cannot hold the full context of every system in your head. **Spawn sub-agents** to investigate: read surrounding code, trace call chains, check what infrastructure the PR should be using, verify tests that should exist. Spawn them in parallel for independent areas. A typical medium PR should spawn 3-8 sub-agents.

## Review Workflow

The workflow is structured as **investigate first, then verify**. Steps 1-5 are active investigation — you trace flows, follow data, and experience the change. Steps 6-10 are verification — you check against checklists, BC guidelines, and existing discussion.

### Step 1: Understand Context

Before reviewing, build understanding of what the PR touches and why:
1. Identify the purpose of the change from title/description/issue
2. Group changes by type (new code, tests, config, docs)
3. Note the scope of changes (files affected, lines changed)
4. Spawn sub-agents to read the unchanged code surrounding each significantly changed file to understand existing patterns and infrastructure

### Step 2: Trace Code Flows

For each significantly changed function, method, or class, **follow the execution path in both directions**. The diff shows what changed — this step reveals what the change affects.

**Trace callers (up the call chain):**
1. Who calls this function? Search the codebase for all call sites.
2. What do callers expect — what arguments do they pass, what return value do they use?
3. If the function's behavior changed, will every caller still work correctly?
4. Are there callers in tests that test the old behavior?

**Trace callees (down the call chain):**
1. What does this function call? Read those functions.
2. What contracts does it depend on — what invariants must hold?
3. If a called function's return type or error behavior changes, does this code handle it?

**Map the blast radius:**
1. If this change is subtly wrong, what symptoms would appear and where?
2. Would the failure be loud (crash, exception) or silent (wrong data, wrong behavior)?
3. How far from this line of code would the symptom manifest?

Spawn sub-agents in parallel to trace each significant change.

### Step 3: Trace Data Flows

For every data shape that crosses a boundary — function to function, module to module, service to service, code to disk/database/API — follow it from creation to final consumption.

**For each data shape, trace the full journey:**

1. **Where is it created?** What validates it at the source?
2. **Where is it stored or transmitted?** What format? What can be lost in translation?
3. **Where is it read back?** What validates it on the way in? Does the reader's schema match the writer's?
4. **Where is it transformed?** Are there intermediate formats, sanitizers, or normalizers that could strip or alter fields?
5. **Where is it consumed?** Does the final consumer interpret every field correctly?

**Specific patterns to investigate:**

- **Type vs runtime validator mismatch** — If the project uses compile-time types AND a runtime validator (Zod, Joi, Pydantic, JSON Schema, etc.), diff the two. Fields in the type but missing from the validator are silently stripped on parse.

- **Dual-path divergence** — If the same data has two code paths (two readers, two writers, a reader and a writer), verify they agree on schema, field priority, and transformation rules. Example: path A does `storedValue ?? derivedValue` but path B does `derivedValue ?? storedValue` — one is wrong.

- **Sanitizer strips data needed later** — If a sanitizer/normalizer processes input before storage, verify it doesn't strip fields that downstream code relies on. Trace whether preservation actually works end-to-end through write → parse → re-write.

- **Writer-reader mismatch** — For every conditional like `if (counter > 0)` or `if (flag)`, trace who sets the value. Are there code paths that should set it but don't?

- **Crash mid-sequence** — If a function performs N sequential operations (file writes, API calls, DB updates), what happens if it crashes after step K? Is state recoverable? Can the operation be safely re-run?

Spawn sub-agents for each data shape to trace in parallel.

### Step 4: Trace User and Consumer Experience

Step back from the implementation. For every changed output surface — API response, CLI output, UI state, event payload, webhook body, error message, log line — read it as the **receiver** would.

**For each output surface:**

1. **Who receives this?** List every downstream consumer — other services, UIs, external tools, human operators reading logs.
2. **What do they see?** Not what the code intends — what the receiver actually gets. A field named `title` will be displayed as a title. Is that actually what this value is?
3. **Can they tell similar things apart?** If the same concept has multiple representations (e.g., a summary used as a fallback for a title), can the consumer distinguish which one it received?
4. **What do they see in transitional states?** Before the value is fully populated — null, empty string, fallback value — is that better or worse than wrong data?
5. **Can they migrate?** If the output shape changes, is there a version signal or deprecation path? Or will consumers silently break?
6. **Would they be surprised?** If you received this output with zero knowledge of the implementation, would any field confuse you? Would you misinterpret anything? Would you be unable to do something you need to do?

Any "yes" to question 6 is a finding.

### Step 5: Deep Review Against Checklist

Go through **every changed line** in the diff and evaluate it against the review checklist in [review-checklist.md](review-checklist.md). This is a systematic sweep to catch anything the flow-tracing steps missed.

### Step 6: Merge Conflict Residue Check

If the PR includes a merge commit or conflict resolution:

1. Diff the PR branch's version of each conflicted file against the base
   branch: `git diff main -- <file>`.
2. For each function, constant, or import present in the base branch but
   absent in the PR branch, verify the removal was intentional.
3. Pay special attention to guard functions, grace periods, timeout
   constants, and safety checks — these are the first casualties of
   `-X theirs` resolution.

### Step 7: Review Existing PR Discussion

Before formulating your review, read what others have already said on the PR:

1. **In CLI mode:** Fetch inline review comments via `gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments` and PR-level reviews via `gh pr view <PR_NUMBER> --json comments,reviews`
2. **In GitHub Actions mode:** Read the `<comments>` section already in the prompt context
3. **Acknowledge valid concerns** — If another reviewer (human or bot, including Copilot) raised a legitimate issue, incorporate or reference it. Do not silently ignore existing discussion.
4. **Disagree explicitly** — If you believe an existing concern is wrong, say so with reasoning. Silence is not disagreement.
5. **Don't duplicate** — Reference existing comments instead of repeating them.

### Step 8: Check Backward Compatibility

Evaluate BC implications per [bc-guidelines.md](bc-guidelines.md). For non-trivial BC questions, spawn a sub-agent to search for existing callers of the modified API.

### Step 9: Edge Case Hunt on High-Risk Files

If the PR touches high-risk code (state machines, parsers, dependency resolution, auth flows, financial calculations, concurrency, or configuration merging), invoke `/edge-case-hunter` on the changed files in those areas. Include the top findings in the Edge Cases section of your review.

Skip this step only if the PR is purely additive (new files, new tests) with no modifications to existing complex logic.

### Step 10: Formulate Review

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
[Round-trip issues, schema mismatches, dual-path inconsistencies,
writer-reader mismatches, crash safety gaps. Only present when
the PR touches persistence, serialization, or data that crosses
module/service boundaries.]

### Consumer Impact
[Issues found by tracing the user/consumer experience — misleading
field names, missing version signals, undistinguishable representations,
broken transitional states. Only present when the PR changes output
surfaces (APIs, events, CLI output, UI state).]

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
