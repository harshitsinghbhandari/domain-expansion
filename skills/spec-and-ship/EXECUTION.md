# Execution: Orchestrator-Driven Build

Read this **after** [IDEATION.md](IDEATION.md) has produced a resolved spec with a batched task breakdown. This half turns that spec into shipped code through Agent Orchestrator (AO), with the human in the loop only at the forks.

## The model

You are the **orchestrator**. You hold the resolved spec and its task breakdown. You do not write the feature code yourself. You:

1. Split the task breakdown into **batches**. A batch is a set of tasks with no dependencies on each other, safe to build in parallel. Any task that needs another task's merged output goes in a later batch.
2. For each batch, spawn **one worker per task** using the standard prompt in [TEMPLATE.md](TEMPLATE.md). Spawn the whole batch at once so the workers run in parallel.
3. Let each worker run autonomously to its definition of done (below). Don't babysit.
4. When a worker reports done, **merge its PR**.
5. When every worker in the batch is merged, start the next batch.
6. Repeat until the breakdown is exhausted.

## Batching rules

- **Parallelize only independent tasks.** Two tasks belong in the same batch only if neither needs the other's code merged first. When unsure, serialize: a wrong parallelization costs a rebuild, a needless serialization costs only time.
- **One worker = one task = one PR.** Keep each task shippable on its own. If a task can't be reviewed and merged independently, it's too big; split it back in the spec.
- A later batch may assume every earlier batch is already merged into the base branch.

## Spawning a batch

Don't cram the whole prompt into `--prompt`. Write the full worker prompt to a file under `/tmp/ao/`, then pass `--prompt` a short brief that points the worker at that file. This keeps the spawn command readable and lets the worker re-read its full brief at any time.

For each task:

```bash
# 1. Write the full prompt (filled-in TEMPLATE.md) to a stable, descriptive path.
mkdir -p /tmp/ao
#    Name it something useful: /tmp/ao/<task-slug>_prompt.md
#    e.g. /tmp/ao/add-rate-limiter_prompt.md

# 2. Spawn with the tracking issue + a one-line brief that points at the file.
ao spawn <issue-or-discussion> --prompt "Work on: <one-line task summary>. Read /tmp/ao/<task-slug>_prompt.md in full before you start; it has your exact task, definition of done, and escalation rules. Follow it."
```

Fill out [TEMPLATE.md](TEMPLATE.md) per task to produce the `/tmp/ao/<task-slug>_prompt.md` file; TEMPLATE.md also gives the short `--prompt` brief. Use a unique `<task-slug>` per task so prompt files never collide. The template encodes the entire done-criteria and escalation loop, so you never re-explain it. Spawn every task in the batch before waiting on any of them.

## The worker contract (enforced by TEMPLATE.md)

Every worker is told to:

1. Do its **main task** against the **tracking issue / discussion**, and open a PR.
2. **Wait until the PR is genuinely done** before reporting:
   - **CI green:** poll every 5 to 10 min until CI passes; fix and re-push on failure.
   - **Review OK** (only if the repo has a reviewer culture): poll review state at 5, 10, and 30 min. If approved, continue. If still no review after 30 min, `inform` that review is stalled.
   - **PR mergeable by CLI:** only once `gh` reports the PR is cleanly mergeable does the worker report **done** to the orchestrator.
3. **Escalate forks, never guess.** On any product or design question the spec didn't settle, the worker runs `inform "<session-id>: <the problem>"`, which webhooks the human on Discord, and waits for the answer instead of picking silently or stalling.

`inform` is the human's personal CLI tool: it fires a Discord webhook so the human can answer a worker mid-run. The worker also reports completion to the orchestrator via `ao send <orchestrator-session-id>` so the orchestrator knows it's time to merge.

## When a worker reports done

1. Re-check the PR is still mergeable (CI can regress after a green run).
2. **Merge it** (`gh pr merge`, or your normal merge path). The orchestrator merges, not the worker, so batch ordering stays under your control.
3. Mark the task complete. When the whole batch is merged, spawn the next batch.

## Your job during a batch

Nothing, until a worker pings you. You answer `inform` Discord pings (product/design forks, stalled reviews) and merge finished PRs. That is the whole loop.

If you find yourself re-explaining the done-criteria to a worker mid-run, the template drifted: fix [TEMPLATE.md](TEMPLATE.md), don't patch it per-spawn. Correct a misdirected worker with `ao send <session-id>` rather than killing and respawning, which throws away its progress.
