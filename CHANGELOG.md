# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Add the `boundary-bug-hunter` skill for aggressive user-flow boundary analysis on a diff or branch. Auto-detects project entry points across TypeScript/JavaScript, Python, Go, Rust, Ruby, Java/Kotlin, .NET, PHP, Elixir, and Dart; traces flows through the diff; identifies seams across six boundary types; demands real round-trip tests; refuses six named deflections; and persists explicit out-of-scope decisions to `.boundary-deferrals.json` for cross-cycle audit.
- Document the new `boundary-bug-hunter` skill in `README.md` and the repository skill manifest.
- Add the `resume-critic` skill for multi-perspective resume critique with five specialized reviewers, anonymous peer review, and an interview-readiness verdict.
- Document the new `resume-critic` skill in `README.md` and the repository skill manifest.
- Add the `resume-rewrite` skill for critique-driven resume rewriting using only existing candidate material.
- Add the `resume-upgrade` skill for critique-driven suggestions on new projects, skills, and proof-building moves.
- Add five new `pr-review` checklist sections to cover gaps that neither `pr-review` nor `boundary-bug-hunter` previously addressed: Observability & Debuggability, Rollout & Migration Safety, Error UX & Operator Affordance, Configuration & Environment, Sensitive Data & Compliance, External Dependencies & Rate Limits.

### Changed
- Restructure `pr-review` Step 3 from inline data-flow tracing to a delegated invocation of `boundary-bug-hunter` in aggressive mode. A `fail` status from boundary-bug-hunter now blocks `Approve` regardless of other findings.
- Replace the `pr-review` "Data Integrity" output section with a "Boundary Verification" section that summarizes the boundary-bug-hunter run and surfaces stale or invalidated deferrals.
- Narrow `pr-review` Step 4 (Consumer Experience) to UX-of-output judgment (field naming, error message actionability, log correlation) — contract-level checks now live in boundary-bug-hunter.
- Replace the `pr-review` review-checklist "Data Flow Integrity" section with a delegation pointer to `boundary-bug-hunter`, eliminating duplicated guidance.
- Tighten `resume-critic` so it requires a target job description or at least a concrete job title, and so every audit explicitly states which job the resume was audited for.
- Make `resume-rewrite` and `resume-upgrade` explicitly depend on a prior matching `resume-critic` output before they can start.
