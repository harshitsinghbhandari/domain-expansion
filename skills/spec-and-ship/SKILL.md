---
name: spec-and-ship
description: >-
  Use when building, adding, or significantly changing any non-trivial feature,
  system, refactor, integration, or migration that will be built out through
  Agent Orchestrator (AO) workers. Also triggers on "interview me", "ask me
  questions first", "help me spec this out", "I want to build X", "let's plan Y
  before we code", or handing over a spec doc / PRD / design brief to start work.
  Skip for bug fixes, one-liners, and cosmetic tweaks.
---

# Spec and Ship

Interview the user into a spec, then orchestrate the build across Agent Orchestrator (AO) workers. Two halves, run in order. Never jump from a vague intent straight to spawning workers.

1. **Ideation.** Drain the ambiguity out of the user's head into a resolved, opinionated spec with a task breakdown, before any code exists. **Read [IDEATION.md](IDEATION.md).**
2. **Execution.** Hand that spec to the AO orchestrator, which spawns one worker per task per batch, waits for each to be genuinely done (CI green, review OK, PR mergeable), merges, and moves to the next batch. **Read [EXECUTION.md](EXECUTION.md).**

Every worker is spawned with the same standardized prompt so the done-criteria and escalation loop never get re-explained. **Use [TEMPLATE.md](TEMPLATE.md)** verbatim when spawning each worker.

## The bet

Building was never the bottleneck. Under-specified intent is, and no amount of execution speed fixes vague. So invert the loop: interrogate the human first, extract a complete spec, then automate the build.

**Automate the execution. Never automate the deciding.**

## The gate

This is deliberate overhead. It pays off only when building the wrong thing would be expensive: a sizeable feature, a new subsystem, a schema change, a public API, a migration, a multi-file refactor. For a bug fix, a one-liner, or a cosmetic tweak, skip both halves and just do the work. (The full gate logic is in [IDEATION.md](IDEATION.md).)
