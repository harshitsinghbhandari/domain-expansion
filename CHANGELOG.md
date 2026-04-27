# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Address [#6](https://github.com/harshitsinghbhandari/domain-expansion/issues/6) — production-readiness gaps in `boundary-bug-hunter`:
  - Per-flow `confidence` label (`static-resolved`, `framework-inferred`, `heuristic`, `runtime-observed`) plus per-language "Known limitations" guidance in `references/entry-point-detection.md`. Heuristic flows are surfaced as such, not silently presented as facts.
  - Optional `--observed-flows <file>` input for runtime-traced call data on opaque codebases.
  - Deterministic top-N flow selection (default 30, configurable via `--top`) with a "Dropped flows" report section listing every dropped flow and why. `--all` and `--scope <glob>` flags for full or filtered analysis. Replaces the prior "ask the user to scope" punt at large diffs.
  - Three additional accepted verification methods alongside round-trip: consumer-driven contract test, recorded-replay test, and seam property test. Full guidance in `references/verification-methods.md`. The skill no longer demands a round-trip when a contract / replay / property test already covers the seam.
  - `BOUNDARY_OVERRIDE` env var / `--override <reason>` flag for emergency wave-throughs, with a loud `.boundary-overrides.log` audit entry, a soft `--max-overrides-per-month` rate limit, and an `audit-overrides` mode that reports the current verification status of every previously-waved-through boundary.
  - `.boundary-verifications.json` cache keyed on `(seamHash, testFileHash)` for faster reruns; invalidates correctly when either hash changes.
  - Trigger-skip heuristic so auto-triggered runs elide diffs with no new function calls, no new imports, no I/O changes, and no schema changes — typo fixes don't burn agent budget.
  - Two additional recognized boundary types: cross-language seam (#7) and event-sourced / CQRS seam (#8). Full guidance in `references/boundary-types.md`.
  - `benchmarks/` scaffold with a recall-corpus / precision-corpus structure and case schema. Corpus collection and runner implementation are tracked as follow-up issues; the spec for the calibration plan now lives in the skill's own SKILL.md.
- Add five new `pr-review` checklist sections to cover gaps that neither `pr-review` nor `boundary-bug-hunter` previously addressed: Observability & Debuggability, Rollout & Migration Safety, Error UX & Operator Affordance, Configuration & Environment, Sensitive Data & Compliance, External Dependencies & Rate Limits.
- Add the `boundary-bug-hunter` skill for aggressive user-flow boundary analysis on a diff or branch. Auto-detects project entry points across TypeScript/JavaScript, Python, Go, Rust, Ruby, Java/Kotlin, .NET, PHP, Elixir, and Dart; traces flows through the diff; identifies seams across eight boundary types; accepts four verification methods; refuses six named deflections; and persists explicit out-of-scope decisions to `.boundary-deferrals.json` for cross-cycle audit.
- Document the new `boundary-bug-hunter` skill in `README.md` and the repository skill manifest.
- Add the `resume-critic` skill for multi-perspective resume critique with five specialized reviewers, anonymous peer review, and an interview-readiness verdict.
- Document the new `resume-critic` skill in `README.md` and the repository skill manifest.
- Add the `resume-rewrite` skill for critique-driven resume rewriting using only existing candidate material.
- Add the `resume-upgrade` skill for critique-driven suggestions on new projects, skills, and proof-building moves.
- Add five new `pr-review` checklist sections to cover gaps that neither `pr-review` nor `boundary-bug-hunter` previously addressed: Observability & Debuggability, Rollout & Migration Safety, Error UX & Operator Affordance, Configuration & Environment, Sensitive Data & Compliance, External Dependencies & Rate Limits.

### Changed
- Remove "Property-based testing" from `boundary-bug-hunter` Out of scope — seam property tests are now an accepted verification method (used at the seam, not as a replacement for unit-level property tests).
- Expand `boundary-status.json` with `was_override`, `flows_dropped`, `verified_breakdown`, `overrides_used`, and `flow_confidence_counts` so CI gates and dashboards can read the new state correctly.
- Restructure `pr-review` Step 3 from inline data-flow tracing to a delegated invocation of `boundary-bug-hunter` in aggressive mode. A `fail` status from boundary-bug-hunter now blocks `Approve` regardless of other findings.
- Replace the `pr-review` "Data Integrity" output section with a "Boundary Verification" section that summarizes the boundary-bug-hunter run and surfaces stale or invalidated deferrals.
- Narrow `pr-review` Step 4 (Consumer Experience) to UX-of-output judgment (field naming, error message actionability, log correlation) — contract-level checks now live in boundary-bug-hunter.
- Replace the `pr-review` review-checklist "Data Flow Integrity" section with a delegation pointer to `boundary-bug-hunter`, eliminating duplicated guidance.
- Tighten `resume-critic` so it requires a target job description or at least a concrete job title, and so every audit explicitly states which job the resume was audited for.
- Make `resume-rewrite` and `resume-upgrade` explicitly depend on a prior matching `resume-critic` output before they can start.
