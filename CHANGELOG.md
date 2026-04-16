# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Add the `resume-critic` skill for multi-perspective resume critique with five specialized reviewers, anonymous peer review, and an interview-readiness verdict.
- Document the new `resume-critic` skill in `README.md` and the repository skill manifest.
- Add the `resume-rewrite` skill for critique-driven resume rewriting using only existing candidate material.
- Add the `resume-upgrade` skill for critique-driven suggestions on new projects, skills, and proof-building moves.

### Changed
- Tighten `resume-critic` so it requires a target job description or at least a concrete job title, and so every audit explicitly states which job the resume was audited for.
- Make `resume-rewrite` and `resume-upgrade` explicitly depend on a prior matching `resume-critic` output before they can start.
