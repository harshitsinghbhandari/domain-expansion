---
name: ao-message-analysis
description: Analyse a person's exported Agent Orchestrator messages ({user}-messages.jsonl) into a private analysis and a redacted shareable one. Use when asked to analyse my AO messages, "what do I ask agents for", "why can't I work hands-off", or to turn a messages.jsonl into analysis.md.
---

# AO message analysis

Input: `{user}-messages.jsonl` in the current folder. "I", "me" and "my" below mean the user who owns the file.

Analyse the user's own messages to Agent Orchestrator agents. The input is `{user}-messages.jsonl` in this folder. Each line is one JSON message I sent (or one that was recorded as sent by me). Produce two files:

- `{user}-analysis.private.md`: the full analysis. Only I will read this.
- `{user}-analysis.shared.md`: a redacted version I can share with the team.

Do not send the file or any part of it to any external service. Do not create subagents, workers, or extra review runs. Work inline.

## Step 1: understand the data

1. Profile the file with a short script: line count, keys per record, value counts for `source`, `harness`, `project_id`, `turn_state`, date range, and text length distribution. Timestamps can mix time zones (UTC and IST), so parse them before sorting.
2. Separate what I typed from what agents wrote on my behalf. Treat these as agent-generated:
   - `ao_chat` messages starting with `[from `, `<task-notification>`, `Reports since`, `AO TASK TITLE`, `<bash-`, or `[Request interrupted`;
   - long, imperative launch briefs written by an orchestrator. These usually come first in a session and read like "Review PR #N... Do not create subagents...".
   - `ao_terminal_launch_prompt` and `ao_terminal_turn` records also have unverified authors. Analyse my own typed messages; use the generated ones only as context (which PRs and tasks existed, how often work was handed over).
3. Write the condensed messages to a scratch file, one per line, in this format: `date | project | session | turn_state | text (first ~300 chars)`. Read ALL of it in chunks. Do not sample. The patterns are in the sequences: an ask, then a correction, then an angry follow-up.

## Step 2: analyse

Answer two questions:

1. **What do I usually ask for?** Rank the categories by approximate count, with one typical example each. Suggested buckets:
   - PR lifecycle: review, approve, merge, conflicts, comments, CI, re-request review
   - worker management: spawn, kill, replace, restore
   - status and visibility ("where are we", open PRs)
   - releases
   - "explain simply"
   - continue, stop, retry nudges
   - product design discussion
   - filing issues and reports

   Add buckets that fit my data better.
2. **Why can't I get work done hands-off?** Rank the causes by how much of my time each costs. For each cause, give:
   - the evidence: dated examples, counts, and the sessions or PRs involved;
   - the underlying mechanism. Is it AO product behaviour, prompt or skill guidance, my own instructions, or something that genuinely needs a human (approvals, secrets, accounts)?
   - recurrences: any rule I stated that was later violated again.

   Then list concrete fixes, ordered by how many of my messages each would remove. Prefer fixes in AO itself (product, prompts, tool-level guards) over "the human should do X".

Also report a few numbers:
- median message length;
- the share of very short messages (40 characters or less);
- frustration messages per month;
- distinct sessions per month;
- the PRs with the most owner changes.

Say which counts are approximate (regex buckets) and what the data can't show. It contains only my side of each conversation.

## Step 3: write the private file

Write `{user}-analysis.private.md` with the full analysis, including short verbatim quotes, session IDs, PR numbers and dates. Keep it skimmable: tables and short sections, no essays.

## Step 4: write the shared file

Start again from the private file and write `{user}-analysis.shared.md` under these rules:

- **Keep:** categories, counts, percentages, ranked causes, mechanisms, fixes, dates, and public PR/issue numbers in the AO repos.
- **Paraphrase, do not quote:** any message that contains personal context, money or pay, opinions about specific people, or anger aimed at a person. Neutral quotes such as "conflicts." or "where are we?" are fine.
- **Remove entirely:**
  - names or handles of other people. Use roles such as "a maintainer" or "an external contributor".
  - secrets, tokens, or token and secret names.
  - IP addresses, hostnames, SSH details, local file paths, keychain entries.
  - links to private docs or sheets, and their IDs.
  - private repo names other than the AO repos.
  - customer or company details, emails, passwords.
  - anything about my personal life, health, finances, or employment.
- **Unsure about something?** Do not decide yourself. Collect it for Step 5.

## Step 5: ask about ambiguous items

Before finishing the shared file, list every item you were unsure about and ask me about all of them in ONE message. Typical ambiguous items:
- cost or money figures, even when they're about work (for example "cost over $100 in tokens");
- PR or issue numbers tied to an incident, a blame-worthy mistake, or a specific person's work;
- internal project, product or codename references;
- dates tied to personal events, or to accounts, limits or access changes;
- workload or count figures that could read as a judgment of me or someone else;
- quotes that are neutral on their own but sensitive in context.

For each item give: the exact text, why it's ambiguous, and your recommendation (keep, reword, or remove). Wait for my answer. Then apply it, and record every item and its decision at the bottom of the private file under "Withheld from shared version". If there are no ambiguous items, say so and skip the question.

## Step 6: verify the shared file

Run a check on `{user}-analysis.shared.md` for:
- IP addresses (`\b\d{1,3}(\.\d{1,3}){3}\b`), `@` emails, `/Users/` and `~/` paths;
- `docs.google.com`, `ssh`, `token`, `PAT`, `password`, `keychain`;
- every person's name or handle that appears in the private file.

Fix every hit, then re-run until it is clean. Report the final check output.

## Output

Reply with only:
1. the paths to both files;
2. the top 3 asks and the top 3 hands-off blockers, one line each;
3. the ambiguous items and what I decided for each.
