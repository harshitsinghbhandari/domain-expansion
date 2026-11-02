---
name: ao-user-time-analyse
description: Collect a private Agent Orchestrator user-message export, analyse an existing export, or run both stages when requested. Use for AO message retrieval and questions about recurring asks, corrections, and human attention cost.
---

# AO user time analysis

This skill has two independent stages. Choose the stage the user requested and load only its instructions.

- **Collect or export messages:** Read [INGEST.md](INGEST.md). Confirm scope, private destination, format, and permission before extracting message bodies. Preserve provenance and coverage gaps. Do not run analysis unless requested.
- **Analyse an existing export:** Read [ANALYSE.md](ANALYSE.md). Use the file the user supplied, even if its path or basename differs from the reference's example. Do not rerun collection or replace the input.
- **Collect and analyse:** Follow INGEST.md first, then ANALYSE.md on the resulting JSONL. Treat unverified launch prompts, Terminal summaries, and agent-generated relay text according to their provenance. Report missing sources and attribution limits before drawing conclusions.

Keep raw messages and private analysis local. The shareable output requires the redaction and ambiguity review in ANALYSE.md. Neither stage authorizes external publication.
