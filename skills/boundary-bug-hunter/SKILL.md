---
name: boundary-bug-hunter
description: Aggressive user-flow and boundary-bug analysis on a diff or branch. Auto-detects entry points, traces flows through changed code, finds every seam (cross-module calls, serialization, file I/O, shared state, schema versioning, network/IPC), and refuses to mark the work complete until each unverified boundary has a real round-trip test or an explicit written out-of-scope record persisted in an audit file. Use whenever the user says "boundary check", "seam check", "round-trip check", "flow boundaries", "user-flow check", "before merge", "is this safe to ship", "pre-merge gate", "boundary bugs", "verify the joins", or asks to validate cross-module joins, producer/consumer contracts, or end-to-end coverage of a change. Also use as a final gate from pr-review on any diff that touches more than one file, module, or process. Be pushy. Most surviving production bugs live at seams, not inside units — if the diff crosses any boundary, this skill almost certainly applies.
---

# Boundary Bug Hunter

Aggressive user-flow boundary analysis on a diff or branch. Discovers every seam the change touches, demands real end-to-end evidence at each one, and refuses to sign off on deflections.

## Why this skill exists

Most bugs that survive review aren't unit-test failures. They're **boundary bugs** — the unit on each side of a seam passes its own tests, but the join is broken. The two halves agree on what they each do; they disagree on what flows between them.

Concrete shapes this skill catches:

- **Producer-consumer drift.** Producer writes shape A, consumer reads shape B. Both unit tests pass. Round-trip silently drops data.
- **Sequenced side effects.** Step 1 mutates shared state, step 2 reads that state. Each step has correct tests for its own effect. The combined flow never runs in tests, so the wrong outcome is invisible.
- **Optional contracts assumed satisfied.** Function returns `null | T`, caller assumes `T`. Each function is tested with its own mocks; nobody runs the real return value through the real caller.
- **Schema version mismatch.** Encoder of version N writes; decoder of version M reads. Each speaks its own version correctly. The version skew is invisible in any single test.
- **Mock-erased downstream symptom.** A mocked side effect can't blow up when called wrong. The mock answers exactly what the test wants — the production implementation never sees the misuse.

These survive because:

- Authors fixate on what they just changed; reviewers naturally read across boundaries.
- Heavy mocking erases the downstream symptom.
- Each side's unit tests pass in isolation by construction.
- "It works on my machine" exercises happy-path data, not the messy values that flow at production scale.

This skill exists to forcibly read **across** the seam, not at it.

## When to use

**Activates explicitly on:**
- "boundary check", "seam check", "round-trip check"
- "flow boundaries", "user-flow check"
- "before merge", "is this safe to ship", "pre-merge gate"
- "boundary bugs", "verify the joins"

**Activates implicitly when:**
- A diff or branch is being reviewed and changes cross more than one file, module, or process.
- `pr-review` invokes it as the final gate (see "Cross-skill integration").
- CI runs the skill in aggressive mode on a PR.

**Required input:** A diff, a branch name, or a working tree against a base ref. If none provided, ask: "Which diff or branch should I check? Provide a PR number, branch name, or `--diff <base>...<head>`."

**Do not run on:**
- Pure documentation changes with no code touched.
- Changes confined to a single function with no callers (rare; verify before skipping).
- Style-only refactors with byte-identical behavior.

## The forcing question

Before the skill signs off on any changed function `f`, it must answer this question concretely:

> **What is the very next thing the next caller of `f` does with `f`'s output, side effect, or thrown exception? Is that next thing covered by a test that uses `f`'s real implementation, not a mock?**

If the answer is "no" or "I don't know", the boundary is **unverified**. The skill names the producer, the consumer, and what's missing.

This question is non-negotiable. Every changed function gets it. Every unverified answer becomes a finding.

## Workflow

### Step 1: Auto-detect the project (project-agnostic)

The skill reads the project, not the user. Detection is required to work without configuration on any codebase that uses a recognizable framework or has explicit `bin` / `exports` / `main` declarations.

In parallel, scan for:

1. **Manifest files** at repo root: `package.json`, `pyproject.toml`, `setup.py`, `Cargo.toml`, `go.mod`, `Gemfile`, `composer.json`, `pubspec.yaml`, `mix.exs`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `*.csproj`, `*.fsproj`, `*.sln`. Multiple manifests = polyglot repo; treat each independently.
2. **Source language(s)** from file extensions and manifest declarations. Support TypeScript/JavaScript, Python, Go, Rust, Ruby, Java/Kotlin, .NET, PHP, Elixir, Dart at minimum.
3. **Test framework** from manifest dev-dependencies and existing test files. Default mapping in [references/test-frameworks.md](references/test-frameworks.md).
4. **Entry-point conventions** for the detected frameworks. Per-framework playbooks live in [references/entry-point-detection.md](references/entry-point-detection.md). Includes Express, Next.js, NestJS, Flask, Django, FastAPI, Click, Cobra, axum, Rocket, Rails, Spring, Quarkus, ASP.NET, plus generic patterns (cron, queues, event subscribers).
5. **Existing test directory conventions** (`tests/`, `__tests__/`, `*_test.go`, `test/`, `spec/`). If absent, fall back to a clear default for the language with a marker file noting where tests should go.

Where the framework is unrecognized and not inferable, **ask once**, then cache the answer in `.boundary-bug-hunter.json` at repo root. Do not invent a framework.

### Step 2: Build user flows that pass through the diff

A **user flow** is a directed call/data chain that starts at an entry point and includes at least one line in the diff.

Compute, in parallel sub-agents per significant change:

- **Forward trace.** From each entry point, the call graph reaching the diff. Stop when the trace cannot reach any changed line.
- **Backward trace.** From each changed line, the consumers that read what the change produces, transitively until an exit shape (HTTP response, log line, persisted record, returned value to an entry point, fired event).
- **Flow set** = forward ∪ backward, restricted to paths that touch the diff.

**Deduplicate** by `(entry-point, exit-shape)` so output is bounded. If two flows share an entry point and end shape but diverge in the middle, treat them as one for the table and call out the branch in prose.

If the diff is large enough that flows are uncountable (>50 deduplicated flows), ask the user to scope to a subset of files, or proceed with a top-N strategy ranked by: (a) number of distinct boundaries on the flow, (b) presence of persistence or external I/O on the flow, (c) test coverage gaps on the flow.

### Step 3: Identify boundaries within each flow

A **boundary** is any seam where one piece of code hands off data, control, or state to another. Six recognized types — full guidance in [references/boundary-types.md](references/boundary-types.md):

1. **Cross-module function call** — call across modules / files / packages.
2. **File I/O** — write here, read there. Include atomic-write/temp-file patterns and lock files.
3. **Serialization / parsing** — JSON / YAML / Protobuf / MessagePack / custom binary in, parsed out. Includes encoder/decoder pairs in any format.
4. **Network / IPC / signals** — HTTP, gRPC, RPC, queues, sockets, signals, OS pipes.
5. **Shared mutable state** — globals, caches, singletons, registries, lock files, environment variables that another process reads.
6. **Schema versioning** — encoder of version N writes; decoder of version M reads. Includes migration boundaries (forward and rollback).

For every boundary on a flow, record:

- **Producer** — `file:line` that writes the data / mutates the state / sends the signal.
- **Consumer** — `file:line` that reads it.
- **Contract** — the shape, keys, types, and invariants the producer claims and the consumer expects. Be specific: don't write "object", write the actual key set with their types.
- **Coverage** — whether any existing test exercises producer **and** consumer **together with real implementations on both sides**, not in isolation, not with mocks erasing the seam.

A boundary is **verified** only when an existing test runs the real producer and the real consumer back-to-back and asserts the round-trip. Anything else — separate unit tests, mocked downstream calls, manual QA, "it works in staging" — is **unverified**.

### Step 4: Demand round-trip tests

For every boundary lacking end-to-end coverage, generate:

- A concrete failing test stub in the project's idiomatic test framework, with exact file path and test name.
- A one-line statement of what the test must assert (e.g., "after producer writes, consumer's read returns non-null and contains field `userId` matching the produced value").
- The exact code stub to copy into the file. Per-language stub templates live in [references/test-frameworks.md](references/test-frameworks.md).

The stub must use **real implementations on both producer and consumer sides**. Where infrastructure makes that impossible (real database, real network), use the project's existing integration-test fixtures, not new mocks.

### Step 5: Refuse the deflections by name

The skill **does not accept** any of the following as a valid reason to skip a round-trip test. When it sees one of these in user replies, PR descriptions, or its own draft reasoning, it names the deflection and refuses:

| # | Deflection | Why it fails |
|---|------------|--------------|
| 1 | "It's a unit-test responsibility." | Boundary bugs live between units. By definition no unit test catches them. |
| 2 | "The contract is documented." | A documented contract that's never executed is a hope, not a check. |
| 3 | "It works in isolation." | Isolation is the failure mode being investigated, not the defense. |
| 4 | "We'd catch it in manual / staging / production." | These find symptoms downstream of the bug. The seam test catches it before it ships. |
| 5 | "The reviewer will catch it." | Reviewers also read units in isolation. The skill exists because they consistently miss seams. |
| 6 | "It's a small change." | Small changes at seams cause large outages. Size is uncorrelated with blast radius at boundaries. |

Full responses for each — to use verbatim when refusing — in [references/deflection-refusals.md](references/deflection-refusals.md).

The refusal is the point. The skill must surface the deflection by name, not absorb it.

### Step 6: Record any explicit out-of-scope decisions

The user may legitimately decide a boundary is out of scope (e.g., the consumer is a third-party system the team doesn't control). In that case the skill **records the deferral** so it remains auditable in the next review cycle.

Append an entry to `.boundary-deferrals.json` at repo root. The exact schema and the seam-hash algorithm (which keeps deferrals valid only as long as the seam structurally matches what was deferred) are in [references/audit-record-format.md](references/audit-record-format.md).

On the next run, the skill loads `.boundary-deferrals.json` and:
- If a current boundary's seam-hash matches a recorded deferral → skip with a one-line note in the report ("deferred on YYYY-MM-DD by `<author>`: `<reason>`").
- If a recorded deferral's seam-hash no longer matches any current boundary → surface as "stale deferral, please remove from `.boundary-deferrals.json`".
- If a current boundary matches a deferral whose seam-hash *changed* (producer or consumer code moved) → invalidate the deferral and require a fresh decision. The seam shifted; the prior judgment is no longer valid.

The user must commit `.boundary-deferrals.json` to the repo. Deferrals not in version control are not deferrals.

## Run modes

### `report` mode (default)

Produces:
- `boundary-report.md` — the human-readable report.
- `boundary-status.json` — machine-readable status for CI gates.

Does not block the agent from continuing other work. Suitable for interactive review.

### `aggressive` mode

Same outputs, plus:
- The agent **must not** mark the surrounding work complete (tasks, PRs, merges) while any boundary is unverified and undeferred.
- If any boundary is unverified at the end of the run, the final agent message includes the literal sentence: `BOUNDARY GATE FAILED: <N> unverified boundaries — see boundary-report.md.`
- For CI integration, the user wires a one-line check on `boundary-status.json` (example: `jq -e '.status == "pass"' boundary-status.json` exits non-zero on fail).

When the user invokes the skill from `pr-review` as a final gate, default to aggressive mode unless told otherwise.

## Output format

### `boundary-report.md`

```markdown
# Boundary Report

**Diff:** <base>...<head> (<commit-count> commits, <file-count> files, +<add>/-<del>)
**Project:** <language(s)> / <test framework(s)>
**Mode:** report | aggressive
**Run date:** YYYY-MM-DD
**Status:** PASS | FAIL (<N> unverified boundaries, <M> deferrals applied, <K> stale deferrals)

## Summary

<1-2 sentence overall assessment.>

Flows discovered: <N>. Boundaries on those flows: <M>. Verified end-to-end: <V>. Unverified: <U>. Deferred (with audit record): <D>.

## User Flow: <entry-point name> → <terminal shape>

<Optional one-line prose describing the flow.>

| # | Boundary | Producer | Consumer | Contract | Verified end-to-end? |
|---|----------|----------|----------|----------|----------------------|
| 1 | <type>   | f1:LL    | f2:LL    | <shape>  | NO                   |

### Missing test (boundary 1)

- **File:** `tests/integration/<name>_test.<ext>`
- **Test name:** `<verb-phrase>`
- **Asserts:** <one-line invariant>

```<lang>
<generated stub in project's idiomatic test framework, runnable as-is>
```

(Repeat the table + per-boundary "Missing test" sections for each flow.)

## Deferrals applied this run

| Boundary | Deferred on | By | Reason | Seam hash |
|----------|-------------|-----|--------|-----------|
| ...      | ...         | ... | ...    | ...       |

## Stale deferrals (please clean up)

| Recorded boundary | Why stale |
|-------------------|-----------|
| ...               | ...       |

## Deflections refused this run

(Only present if the agent encountered one of the named deflections during this analysis.)

| # | Deflection | Where it appeared | Refused with |
|---|------------|-------------------|--------------|
| 1 | "It's a unit-test responsibility." | PR description, line 12 | <one-line response> |

## Final verdict

<PASS or FAIL with a one-line reason. If FAIL, the literal sentence: BOUNDARY GATE FAILED: <N> unverified boundaries — see this report.>
```

### `boundary-status.json`

```json
{
  "status": "pass" | "fail",
  "unverified_count": 0,
  "deferred_count": 0,
  "stale_deferral_count": 0,
  "flows_analyzed": 0,
  "boundaries_total": 0,
  "diff_base": "<sha>",
  "diff_head": "<sha>",
  "run_at": "<ISO-8601>"
}
```

## Cross-skill integration

- **`pr-review`** — When `pr-review` finishes its other steps and the diff touches more than one file, invoke this skill in aggressive mode as a final gate. A `pr-review` Approve recommendation with unverified boundaries is contradictory and must be downgraded.
- **`edge-case-hunter`** — Complementary, not duplicative. `edge-case-hunter` finds input-side edges within a function (nulls, empties, boundaries). This skill finds seam-side edges between functions (round-trip drift, missing combined coverage). Run both on high-risk diffs.
- **`test-coverage-audit`** — `test-coverage-audit` audits the existing test suite globally. This skill audits whether the *specific seams introduced or modified by the diff* have tests. Run this skill on the diff; run `test-coverage-audit` on the suite.

If the user invokes any of those skills, mention this one is a relevant follow-up when the diff crosses boundaries.

## Quality bar

These principles override convenience and apply to every run:

1. **If the boundary exists, surface it.** Do not rationalize away a finding because the producer "obviously" matches the consumer. Obviousness is the failure mode.
2. **Default to skepticism.** A boundary is unverified until a real round-trip test proves otherwise. CI green, "looks fine", and prior reviewer approvals are not evidence.
3. **Fetch everything.** Read the producer file. Read the consumer file. Read the existing tests for both. Read the diff against the base. Do not infer from one source what three sources can confirm.
4. **Validate, don't trust.** Mocks, fixtures, and prior tests are claims. The round-trip test is the validation. Treat anything else as unverified.
5. **Stay project-agnostic.** No hardcoded repo names, language preferences, or framework assumptions. The skill must work on any codebase the manifest detection recognizes.
6. **The report goes under the user's name.** Every approval, every "verified", every deferral acceptance reflects on them. Be specific. Be skeptical. Be willing to deliver an inconvenient FAIL.

## Acceptance criteria (from the originating issue)

The skill is correct when:

- On a synthetic test repo seeded with a known producer/consumer shape mismatch, the skill flags the boundary **and** produces a working failing-test stub before any human reads the code.
- On a synthetic test repo seeded with a two-step state mutation where step 1 + step 2 have unit tests but no combined test, the skill flags the missing combined flow.
- On a clean diff with no boundary changes, the skill produces zero false positives.
- The skill detects test framework and language for at least: TypeScript/JavaScript (Vitest / Jest / Mocha), Python (pytest / unittest), Go, Rust, Ruby (RSpec / minitest), Java/Kotlin (JUnit), .NET (xUnit / NUnit) — without manual configuration.
- An aggressive-mode CI run on a PR with an unverified boundary surfaces the literal `BOUNDARY GATE FAILED` sentence and `boundary-status.json` reports `"status": "fail"`.
- When the user provides a written out-of-scope reason for a boundary, the skill records it in `.boundary-deferrals.json` with the seam hash so the deferral is auditable in the next review cycle.

## Out of scope

- Replacing unit tests. This is additive.
- Mutation testing.
- Property-based testing.
- Full browser / network end-to-end. The seams the skill hunts are within-process and file-system seams reachable from the diff.
- Style / lint / type concerns. Those have other tools.
- Performance regressions. Use a profiler or benchmark harness.

## Example trigger phrases

- "Boundary check this branch."
- "Run a seam check before I merge."
- "Are the round-trips covered?"
- "Verify the user flows touched by this diff."
- "Boundary-bug check on PR 123."
- "Is this safe to ship?"
- "Pre-merge gate."
- "Find boundary bugs in the new payments code."
