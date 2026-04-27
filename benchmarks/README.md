# Boundary Bug Hunter — Benchmarks

Calibration suite for the `boundary-bug-hunter` skill. Synthetic seeded tests verify happy-path behavior; real-world precision and recall come from running the skill against historical PRs.

This directory is the **scaffold**. Corpus collection is tracked as a follow-up issue.

## Suites

### `recall-corpus/`

Real merged PRs from open-source projects where reviewers caught a boundary bug. The skill is run on the **pre-fix commit** and is expected to flag the same boundary the reviewer flagged.

A miss is a blocking regression for a release.

#### Entry format

Each entry is a directory `recall-corpus/<short-id>/` containing:

```
recall-corpus/
  <short-id>/
    case.json       # metadata (see schema below)
    expected.json   # the boundary the skill must surface
    README.md       # short prose describing the bug, the fix, the lesson
```

`case.json`:

```json
{
  "id": "<short-id>",
  "source": "<owner>/<repo>#<pr-number>",
  "preFixCommit": "<sha>",
  "fixCommit": "<sha>",
  "boundaryType": "cross-module-call" | "file-io" | "serialization" | "network-ipc" | "shared-state" | "schema-version" | "cross-language" | "event-sourced",
  "language": "typescript" | "python" | "go" | "rust" | "ruby" | "java" | "kotlin" | "csharp" | "php" | "elixir" | "dart" | "polyglot",
  "addedToCorpusAt": "<ISO-8601>",
  "tags": ["regression", "round-trip", "schema-mismatch", "version-skew", ...]
}
```

`expected.json`:

```json
{
  "boundaries": [
    {
      "producer": { "file": "<path>", "function": "<name>" },
      "consumer": { "file": "<path>", "function": "<name>" },
      "contract": "<short description>"
    }
  ]
}
```

#### Seed corpus

The corpus starts empty. Initial seeds to add (from the originating skill design and follow-up review):

- Any PR with a `boundary` / `regression` / `round-trip` / `schema mismatch` / `version skew` tag in commit messages or review threads.
- The two PRs referenced in the originating skill design — the producer/consumer mismatch case and the sequenced-state-mutation case.

A high-quality recall corpus needs **at least 20 cases across at least 4 boundary types and 4 languages** before precision/recall numbers from it are statistically meaningful. Seed it incrementally.

### `precision-corpus/`

Real merged PRs that touched cross-boundary code but introduced no boundary bugs (no follow-up fix, no incident). The skill is expected to stay quiet on these — produce zero unverified-boundary findings or label every finding with high enough confidence that an experienced reviewer would agree.

False positives degrade trust faster than false negatives degrade safety. Treat any spurious finding as a blocker.

#### Entry format

Same shape as `recall-corpus/<id>/`, but `expected.json` is:

```json
{
  "boundaries": [],
  "acceptableFindings": "<rationale: zero, OR explicit list of low-confidence findings the skill is allowed to surface so long as they are clearly labeled>"
}
```

## Running the benchmark

(To be implemented as a follow-up. Suggested shape:)

```bash
node benchmarks/run.mjs --suite recall   # → benchmarks/results-recall-<sha>.json
node benchmarks/run.mjs --suite precision # → benchmarks/results-precision-<sha>.json
node benchmarks/run.mjs --release v0.x    # combined release report → benchmarks/results-v0.x.json
```

The runner:

1. Clones each case into a temp dir at the `preFixCommit`.
2. Runs `boundary-bug-hunter` in aggressive mode against the pre-fix commit's diff.
3. Compares the run's findings against `expected.json`.
4. Records: detected boundaries, missed boundaries, spurious boundaries, runtime, confidence labels.

Results JSON includes per-case verdicts and aggregate precision / recall:

```json
{
  "skill_version": "<git-sha>",
  "suite": "recall" | "precision",
  "cases_run": 0,
  "true_positives": 0,
  "false_positives": 0,
  "false_negatives": 0,
  "true_negatives": 0,
  "precision": 0.0,
  "recall": 0.0,
  "by_boundary_type": { "cross-module-call": { "tp": 0, "fp": 0, "fn": 0 }, "...": {} },
  "by_language": { "typescript": { "tp": 0, "fp": 0, "fn": 0 }, "...": {} },
  "ran_at": "<ISO-8601>"
}
```

## CI

A GitHub Actions workflow (to be added) runs both suites on every PR that modifies files under `skills/boundary-bug-hunter/`. A regression in either precision or recall blocks merge.

## Status

- [x] Directory structure
- [x] Case + expected schema
- [ ] Recall corpus seeded (follow-up issue)
- [ ] Precision corpus seeded (follow-up issue)
- [ ] Runner implementation (follow-up issue)
- [ ] CI workflow (follow-up issue)
