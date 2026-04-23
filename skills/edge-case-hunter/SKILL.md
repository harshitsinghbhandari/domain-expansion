---
name: edge-case-hunter
description: Exhaustive edge-case review that traces branching paths and boundary conditions in a diff or code sample and reports only unhandled cases. Use when the user asks to hunt edge cases, boundary conditions, missing guards, or wants an edge-case-focused review.
---

# Edge Case Hunter

An aggressive edge-case discovery skill that systematically hunts production break points in code. Ignores happy paths entirely. Assumes nominal behavior tests already exist. Surfaces only the highest-risk unhandled edge cases with ready-to-use test code.

## Philosophy

**100% edge cases only.** Happy-path tests are assumed to exist and are explicitly ignored. This skill is deliberately thorough and paranoid — it hunts every realistic production break point: nulls, empties, malformed data, timeouts, auth expiry, partial failures, race-prone sequences, and boundary conditions.

The goal is not comprehensive test coverage. The goal is catching the bugs that will page you at 3am.

**If you found it, report it.** Do not rationalize away a finding as "unlikely" or "the developer probably knows." Your report goes under the user's name — every missed edge case reflects on them personally.

## Activation Rules

**Explicit triggers only.** This skill activates ONLY when the user explicitly mentions:
- "hunt edge cases"
- "edge case review"
- "find edge cases"
- "boundary conditions"
- "missing guards"
- "what could break"
- "edge-case-focused review"

**Required input:** User must provide source code files, a directory, or reference changed files in a branch/PR. If no target is provided, ask: "What code would you like me to hunt edge cases in? Provide a directory path, file(s), or branch name."

**Languages supported:** Auto-detected from file extensions. Works with any language; depth is strongest for Python, TypeScript, Go, Java, and Rust.

## Constraints

### LOC Limit

- **Default limit:** 15,000 lines of code per scan
- Automatically counts LOC (ignoring `node_modules`, `__pycache__`, `.git`, test files, etc.)
- If target exceeds limit, respond immediately:
  > "Directory too large (XX kLOC). Please narrow to a subdirectory, or I can suggest the tightest sub-directories that stay under the limit."
- User can override with explicit request (e.g., "scan anyway" or "use 25k limit")

### Scope

- Single directory or file set per invocation
- No CI integration — purely manual/on-demand
- Focuses on code the user points at, not the entire repository

## Risk Scoring

Every potential edge case is scored by **Risk = Impact × Probability**.

### Impact (1-10): Blast Radius in Production

| Score | Meaning |
|-------|---------|
| 9-10 | User data corruption, security hole, total outage, money loss |
| 7-8 | Crashes request pool, wrong response to customer, background job failure |
| 5-6 | Degraded performance, silent wrong answer, retry storm |
| 3-4 | Cosmetic issues, logging noise, monitoring false positives |
| 1-2 | Negligible, purely theoretical |

### Probability (1-10): Likelihood in Production

Based on:
- **Input surface:** User input (high), external API (high), DB/queue (medium), internal config (low)
- **Control-flow complexity:** Cyclomatic complexity, nested branches, external calls
- **Dependency risk:** Rate limits, auth expiry, network calls, shared mutable state
- **Historical signals:** Bug-fix commits, TODOs mentioning "edge" or "TODO"
- **Combinatorial filter:** If >3 independent unlikely conditions required, auto-cap at 2

### Cutoff

- **Aggressive mode (default):** Include all edges with Risk ≥ 18
- Only the **top 8-15** highest-risk edges make the final report
- Lower-risk edges are dropped, not hidden — the skill explicitly says "47 potential edges found, 12 surfaced"

## Workflow

### Step 1: Validate Target

1. Check if target exists and is accessible
2. Count LOC (excluding noise directories)
3. If over limit, ask user to narrow scope or confirm override
4. Detect language(s) from file extensions

### Step 2: Deep Code Analysis

Parse the target code and identify:
- All functions with branches, external calls, or database/queue touches
- Input sources (user, API, DB, config)
- Error handling patterns (or lack thereof)
- Assumptions baked into the code (null checks missing, type coercion, etc.)

For each function, systematically generate edge candidates using:
- **Boundary analysis:** Empty inputs, max values, negative numbers, unicode edge cases
- **Failure injection:** Network timeout, auth expiry, partial writes, rate limits
- **State corruption:** Concurrent access, cache invalidation, stale reads
- **Type confusion:** null/undefined, wrong type, missing fields

### Step 2b: Data Flow Edge Cases

In addition to input-driven edge cases, trace data-flow edge cases. These tend to produce high-Impact, high-Probability edges because they involve persistence and external systems — the bugs that pass CI and break in production.

**1. Schema round-trip breaks**

Find every place data is serialized and deserialized (written to disk, database, config, API). For each:
- Does the read schema accept everything the write schema produces?
- Are there values the write path can produce that the read path silently drops or rejects?
- If both compile-time types AND runtime validators exist (TypeScript + Zod, Python + Pydantic), are they in sync? Missing runtime enum values cause silent data loss on re-read.

**2. Stale state after transformation**

Find every place state is stored, cached, or derived. For each:
- What happens if the underlying truth changes after the state was stored? Does the cached state update, or does it go stale and cause incorrect behavior?
- After a transformative action (restore, migrate, archive, role change), is the stored state updated to match the new reality? Or does pre-action state linger and cause problems on next read?

**3. Partial operation states**

Find every multi-step mutation (multiple writes, moves, renames, network calls). For each:
- What does the system look like if it crashes between step N and N+1?
- Is re-run safe (idempotent), or does it produce duplicates/corruption?
- Are temp files cleaned up, or do they accumulate indefinitely?

**4. External tool desync**

Find every interaction with external tools (git, tmux, Docker, k8s). For each:
- What happens if the code's view of the world diverges from the tool's? (Moved files git doesn't know about, tmux sessions the code renamed but the tool didn't, Docker containers the code stopped but the state file still says running.)
- After moving/renaming a resource the tool tracks, is the tool's internal state repaired?

**5. Dual-path inconsistencies**

Find cases where the same conceptual data has two code paths (two readers, two writers, two derivation functions). For each:
- Do they produce the same result for the same input?
- Do they agree on priority (stored vs derived, cache vs fresh)?
- Counter/flag mismatches: does every code path that should increment a counter actually increment it?

### Step 3: Score and Rank

For each edge candidate:
1. Assess Impact (1-10)
2. Assess Probability (1-10)
3. Calculate Risk = Impact × Probability
4. Filter to Risk ≥ 18
5. Sort by Risk descending
6. Take top 8-15

### Step 4: Generate Report

Produce `edge-case-report.md` with the structure defined below.

## Output Format

Generate `edge-case-report.md` with this exact structure:

```markdown
# Edge Case Hunter Report

**Target:** [directory/file path]
**LOC scanned:** [X,XXX lines]
**Language:** [Python / TypeScript / Mixed]
**Scan date:** [YYYY-MM-DD]

## Summary

Found **[N] potential edge cases**.
Top **[M] highest-risk** edges surfaced below (all Risk ≥ 18).
No happy-path tests were generated.

## Risk Table

| # | Edge Case | Impact | Prob | Risk | Location |
|---|-----------|--------|------|------|----------|
| 1 | [description] | [1-10] | [1-10] | [score] | `[file:line]` |
| 2 | ... | ... | ... | ... | ... |

## Detailed Edge Cases

### 1. [Edge Case Name] (Risk [score])

**Where it fails:**
[Exact function/method and what assumption breaks]

**Code path that breaks:**
```[language]
# [file:line]
[relevant code snippet]
```

**Why this is realistic:**
[Explain probability — has this happened in similar systems? What triggers it?]

**Suggested edge-case test:**
```[language]
[ready-to-copy test code using the project's test framework]
```

---

### 2. [Next Edge Case Name] (Risk [score])
...

## Edges Considered but Deprioritized

[Optional section listing a few edges that scored Risk < 18 with brief explanation of why they were dropped, e.g., "requires 4+ unlikely conditions to align"]
```

## Standards for Analysis

### Be Aggressive but Realistic

- Hunt every realistic break point, but don't fabricate scenarios that require absurd preconditions
- "100 non-normal conditions to align" = Risk crushed to near-zero, dropped from report
- Prefer edges that have actually happened in similar systems over purely theoretical ones

### Reference Everything

- Every edge case must cite exact file path and line number
- Include the actual code snippet that breaks
- Explain the specific input/state that triggers the failure

### Provide Runnable Tests

- Generate test code matching the project's test framework (auto-detect pytest, unittest, Jest, etc.)
- Tests should be copy-paste ready
- Include setup/mocking for external dependencies
- Assert the specific failure mode (exception type, error message, status code)

### Don't Repeat Other Skills

- This skill does NOT audit code quality (use code-quality-audit)
- This skill does NOT audit test coverage (use test-coverage-audit)
- This skill does NOT review PRs holistically (use pr-review)
- This skill ONLY finds edge cases

## Invocation Examples

```
/edge-case-hunter path/to/my-service/core/
```

```
/edge-case-hunter --branch feature/payments
```

```
User: "Hunt edge cases in the orders module"
Agent: [scans orders/ directory, produces report]
```

```
User: "What edge cases am I missing in auth.py?"
Agent: [scans auth.py specifically, produces focused report]
```

## Quality Assurance

- Do not invoke this skill unless explicitly triggered
- Do not generate happy-path tests — assume they exist
- Do not auto-fix the code — only report and suggest tests
- If LOC limit exceeded, ask before proceeding
- If no edge cases found with Risk ≥ 18, explicitly say so (rare but possible for very robust code)

## Example Trigger Phrases

- "Hunt edge cases in the payments module"
- "What edge cases am I missing in this PR?"
- "Find boundary conditions in auth.py"
- "What could break in this code?"
- "Edge case review for services/user/"
- "Look for missing guards in the API handlers"
