# Worker Spawn Template

Two pieces per worker:

1. **The prompt file:** written to `/tmp/ao/<task-slug>_prompt.md`. Holds the full task, definition of done, and escalation rules.
2. **The spawn brief:** a one-liner passed to `ao spawn --prompt`. It just summarizes the task and points the worker at the prompt file.

Spawn one worker per task. Use a unique `<task-slug>` per task so prompt files never collide. Do not trim the "Definition of done" or "If you get stuck" sections from the prompt file; they are the point of the template.

---

## Part 1: Prompt file (`/tmp/ao/<task-slug>_prompt.md`)

Fill the `{{...}}` placeholders, write the result to `/tmp/ao/<task-slug>_prompt.md`. Everything below the line is the file's content.

---

## Your task

{{MAIN_TASK}}

## Tracking

GitHub issue / discussion: {{ISSUE_OR_DISCUSSION_URL}}

Reference it in your PR description (for example "Closes #NN", or link the discussion).

## Definition of done (do NOT report done before all of this holds)

Open your PR early, then drive it to a clean merge. Use your own AO session id wherever a message needs it; it's in the `AO_SESSION_ID` env var (run `echo $AO_SESSION_ID`).

1. **CI must be green.**
   - After you push, poll CI every 5 to 10 minutes.
   - If CI fails, read the failure, fix it, push again, keep polling.

2. **Reviewer must be OK** (only if this repo expects human review; if it doesn't, skip to step 3).
   - Poll the review state at roughly 5 min, 10 min, and 30 min after the PR is review-ready.
   - If approved: continue.
   - If changes are requested: address them, push, and restart this step.
   - If there is still no review after 30 min: run
     `inform "$AO_SESSION_ID: PR for {{PR_OR_TASK_REF}} has been awaiting review for 30+ min"`
     and keep waiting.

3. **PR must be cleanly mergeable by CLI.**
   - Confirm `gh pr view` / `gh pr merge --dry-run` reports the PR is mergeable: no conflicts, all required checks passed.
   - Only when CI is green AND (review is OK or not required) AND the PR is CLI-mergeable are you actually done.

4. **Report done, then stop.** Tell the orchestrator you are finished:
   `ao send {{ORCHESTRATOR_SESSION_ID}} "done: {{PR_OR_TASK_REF}} is green, reviewed, and mergeable"`
   Do NOT merge the PR yourself. The orchestrator merges it and starts the next batch.

## If you get stuck on a product or design decision

Do NOT guess, and do NOT stall silently. Run:

`inform "$AO_SESSION_ID: <describe the product/design question and the options you see>"`

This pings the human on Discord. Wait for their answer, then proceed with their decision.

---

## Part 2: Spawn brief (the `--prompt` value)

Keep it to one or two sentences. Summarize the task, then point at the prompt file:

```
Work on: <one-line task summary>. Read /tmp/ao/<task-slug>_prompt.md in full before you start; it has your exact task, definition of done, and escalation rules. Follow it.
```

Full spawn:

```bash
ao spawn {{ISSUE_OR_DISCUSSION}} --prompt "Work on: <one-line task summary>. Read /tmp/ao/<task-slug>_prompt.md in full before you start; it has your exact task, definition of done, and escalation rules. Follow it."
```

## Placeholders to fill

- `{{MAIN_TASK}}`: the specific task description for this worker, from the spec's task breakdown.
- `{{ISSUE_OR_DISCUSSION_URL}}` / `{{ISSUE_OR_DISCUSSION}}`: the GitHub tracking issue or discussion for this task.
- `{{ORCHESTRATOR_SESSION_ID}}`: your (the orchestrator's) AO session id, so the worker can report completion back to you.
- `{{PR_OR_TASK_REF}}`: a short handle for the PR/task (the task name up front; the PR number once it exists).
- `<task-slug>`: a unique, descriptive slug for this task, used in the prompt file path.

The worker fills its own session id at runtime from `$AO_SESSION_ID`, so you don't need to know it before spawning.
