---
name: llm-forge
description: Run existing work through 5 specialist craftspeople who each produce an improved version, then peer-review and synthesize the best into a single improved artifact. Use when the user says "forge this", "improve this", "make this better", "level this up", "refine this", or asks for multi-angle improvement on code, copy, strategy, plans, designs, or any artifact where the current version works but could be significantly better. Do NOT use for decisions (use llm-council), simple edits, or creation from scratch.
---

# LLM Forge

An improvement framework that takes existing work, runs it through 5 independent craftspeople with different specializations, has them peer-review each other's improvements, then synthesizes the best elements into one final, superior version.

**The difference from LLM Council:** Council debates and recommends. Forge rewrites and delivers. You walk away with the improved thing, not just opinions about it.

## When to Use

The forge is for artifacts where **the current version works but could be significantly better**.

**Good forge inputs:**
- "Forge this: [landing page copy]" — tighten, restructure, make it convert
- "Make this better: [API design / function / module]" — cleaner, more robust, better structured
- "Improve this: [strategy doc / proposal / plan]" — sharpen the thinking, close gaps, strengthen the argument
- "Level this up: [email sequence / pitch deck outline]" — more compelling, better flow, fewer weak spots
- "Refine this: [system prompt / skill definition]" — more precise, fewer edge cases, better coverage

**Bad forge inputs (use something else):**
- "Should I do X or Y?" (decision — use llm-council)
- "Write me a landing page from scratch" (creation task, not improvement)
- "Fix this typo" (trivial edit, just do it)
- "Review this PR" (review task — use pr-review)

## The Five Craftspeople

Each craftsperson produces an **actual improved version**, not just feedback. They create natural tensions with each other.

### 1. The Surgeon
Cuts ruthlessly. Removes bloat, redundancy, filler, and anything that doesn't earn its place. If something can be said in fewer words or fewer lines, says it. If a section adds no value, deletes it. The friend who crosses out half your draft and it reads better.

### 2. The Architect
Restructures for clarity, flow, and scalability. Reorders sections so ideas build on each other. Regroups related concepts. Adds scaffolding where the reader or user gets lost. Doesn't care about word count (that's the Surgeon's job). Cares about whether the structure carries the weight of the content.

### 3. The Sharpener
Precision specialist. Tightens language, sharpens logic, eliminates ambiguity. Replaces vague claims with specific ones. Replaces passive voice with active. Makes every word, every line, every parameter pull its weight. The difference between "pretty good" and "airtight."

### 4. The Humanizer
Makes it resonate with real humans. Adds voice, empathy, concrete examples, and real-world grounding. Catches jargon that excludes people. Catches abstractions that lose people. If the Sharpener makes it precise, the Humanizer makes it land. Tests everything against: "Would a real person actually understand and care about this?"

### 5. The Stress-Tester
Finds what breaks under pressure, then fixes it. Edge cases, adversarial reads, failure modes, missing error handling, unstated assumptions, logical gaps. Doesn't just point out problems — produces a version where the problems are solved. The craftsperson who ships the thing that survives contact with reality.

**Why these five:** Three natural tensions:
- Surgeon vs Architect (cut down vs build up)
- Sharpener vs Humanizer (precision vs resonance)
- Stress-Tester sits in the middle keeping everyone honest — a quality gate the other four must survive

## Workflow

### Step 1: Context Enrichment & Framing

Before framing, scan the workspace for relevant context:
- `CLAUDE.md` or `claude.md` in project root (style preferences, constraints)
- Any `memory/` folder (audience profiles, past decisions)
- Files the user explicitly referenced
- Previous forge transcripts (to avoid re-forging the same ground)

Use `Glob` and quick `Read` calls. Don't spend more than 30 seconds. Look for 2-3 files that give craftspeople specific, grounded context.

**Frame the artifact** with:
1. The artifact itself (full text, code, or content)
2. What it is and what it's for (function, audience, goal)
3. Key context from workspace files (style guides, constraints, audience)
4. What matters most (speed? clarity? conversion? robustness? — if unstated, infer from the artifact type)

If the artifact is too vague or too large (>2000 words / >200 lines of code), ask ONE clarifying question or ask the user to scope it, then proceed.

### Step 2: Convene the Forge (5 Sub-agents in Parallel)

Spawn all 5 craftspeople simultaneously using the Agent tool. Each gets:

**Sub-agent prompt:**
```
You are [Craftsperson Name] in an LLM Forge.

Your specialization: [craftsperson description from above]

A user has brought this artifact to the forge for improvement:
---
[framed artifact with context]
---

Produce an IMPROVED VERSION of this artifact from your perspective. Do not just critique — deliver the better version. You may restructure, rewrite, cut, add, or transform as your specialization demands.

After your improved version, add a brief "Changes Made" section (3-5 bullet points) explaining what you changed and why.

Lean fully into your specialization. The other craftspeople will cover the angles you're not covering. Don't try to do everything — do your thing exceptionally well.

Keep your improved version roughly the same scope as the original (don't halve or double it unless your specialization demands it — e.g., the Surgeon may legitimately cut it in half).
```

### Step 3: Peer Review (5 Sub-agents in Parallel)

Collect all 5 improved versions. **Anonymize as Version A through E** (randomize mapping to prevent positional bias).

Spawn 5 reviewer sub-agents. Each sees the original artifact, all 5 improved versions, and answers:
1. Which version is the strongest overall improvement and why? (pick one)
2. What is the single best change across ALL versions that the final synthesis must keep?
3. Which version introduced a regression — something that was better in the original? What was it?
4. What did ALL versions fail to improve that still needs work?

**Reviewer prompt:**
```
You are reviewing the outputs of an LLM Forge. Five craftspeople independently improved this artifact:

ORIGINAL:
---
[original artifact]
---

Here are their anonymized improved versions:

**Version A:**
[version]

**Version B:**
[version]

**Version C:**
[version]

**Version D:**
[version]

**Version E:**
[version]

Answer these four questions. Be specific. Reference versions by letter.
1. Which version is the strongest overall improvement? Why?
2. What is the single best change across ALL versions that the final version must keep?
3. Which version introduced a regression — something that was better in the original? What was it?
4. What did ALL five versions fail to improve that still needs work?

Keep your review under 250 words. Be direct.
```

### Step 4: Master Forger Synthesis

One agent gets everything: original artifact, all 5 improved versions (de-anonymized), and all 5 peer reviews.

**Master Forger prompt:**
```
You are the Master Forger. Your job is to synthesize the work of 5 craftspeople and their peer reviews into one final, superior version of the artifact.

ORIGINAL ARTIFACT:
---
[original artifact with context]
---

CRAFTSPEOPLE VERSIONS:

**The Surgeon's Version:**
[version + changes]

**The Architect's Version:**
[version + changes]

**The Sharpener's Version:**
[version + changes]

**The Humanizer's Version:**
[version + changes]

**The Stress-Tester's Version:**
[version + changes]

PEER REVIEWS:
[all 5 peer reviews]

Produce the final forged artifact using this exact structure:

## The Forged Version
[The final, improved artifact. Take the best elements from each craftsperson. Where craftspeople conflict (e.g., Surgeon cut something the Architect restructured), make a judgment call and pick the stronger choice. Where peer reviewers flagged regressions, preserve the original's strength. Where all versions missed something, fix it yourself.]

## What Changed and Why
[A concise summary of the major improvements, organized by theme not by craftsperson. Explain the key judgment calls where craftspeople conflicted.]

## What the Forge Couldn't Fix
[Honest list of remaining weaknesses that need human input — missing information, strategic decisions only the user can make, constraints the forge couldn't resolve. If nothing, say "None identified."]

## Improvement Summary
[A one-line statement of the single biggest improvement the forge achieved.]

Be direct. Deliver the best possible version. Don't hedge or water down strong improvements to avoid conflict between craftspeople. The user brought this to the forge because they want it to come back better.
```

### Step 5: Generate HTML Report

Create `forge-report-[timestamp].html` with inline CSS. Structure:

1. **The artifact description** at the top (what it is, what it's for)
2. **The Forged Version** prominently displayed — this is the main deliverable
3. **Before/After comparison** — side-by-side or inline diff showing key changes
4. **What Changed and Why** section
5. **What the Forge Couldn't Fix** section
6. **Collapsible sections** for each craftsperson's full version (collapsed by default)
7. **Collapsible section** for peer review highlights
8. **Footer** with timestamp

Style: white background, subtle borders, system sans-serif font, soft accent colors for each craftsperson. Professional workshop report look. Use a teal/forge-blue accent color to distinguish from council reports.

**Open the HTML file** after generating so the user sees it immediately.

### Step 6: Save Transcript

Save `forge-transcript-[timestamp].md` with:
- Original artifact
- Framed context
- All 5 craftspeople versions with their change notes
- All 5 peer reviews (with anonymization mapping revealed)
- Master Forger's full synthesis

## Output Files

Every forge session produces:
```
forge-report-[timestamp].html       # visual report with the forged version front and center
forge-transcript-[timestamp].md     # full transcript for reference
```

## Important Rules

- **Always spawn all 5 craftspeople in parallel.** Sequential spawning wastes time and lets earlier versions influence later ones.
- **Each craftsperson produces an actual improved version, not just feedback.** Critique without a rewrite is not forging. If a craftsperson can't improve the artifact from their angle, they must say so explicitly and return the original unchanged with an explanation.
- **Always anonymize for peer review.** If reviewers know which craftsperson produced which version, they'll defer to certain specializations instead of evaluating on merit.
- **The Master Forger can reject any craftsperson's changes.** If an "improvement" actually makes things worse (e.g., the Surgeon cut something load-bearing), the Master Forger restores the original and explains why.
- **The Master Forger can go beyond the five versions.** If all five missed something that the peer reviewers flagged, the Master Forger fixes it directly rather than leaving it broken.
- **Don't forge trivial artifacts.** If the input is a one-liner or a typo fix, just improve it directly.
- **The forged version is the deliverable.** The report is context. Most users will copy the forged version and move on. Make sure it stands alone without needing the report to make sense.
- **Output goes under the user's name.** If they ship the forged version, their reputation is on the line. The forge owes them quality, not quantity.
- **Preserve intent.** The forge improves execution, not direction. If the user wrote a casual blog post, don't forge it into a formal whitepaper. Match the original's register and purpose unless told otherwise.

## Forge Modes

The user can optionally request a focus mode to weight certain craftspeople more heavily:

- **"forge lean"** — Surgeon and Sharpener get double weight in synthesis (cut the fat)
- **"forge rich"** — Architect and Humanizer get double weight (build it out, make it resonate)
- **"forge bulletproof"** — Stress-Tester gets double weight, all versions must survive stress-testing before synthesis

If no mode is specified, all craftspeople are weighted equally.

## Example Session

**User:** "Forge this: Here's my cold email template for reaching out to potential clients..."

**The Surgeon:** Cuts the email from 200 words to 90. Removes the company history paragraph, the "I hope this finds you well" opener, and two redundant value props. Result: a tight, scannable email that respects the reader's time.

**The Architect:** Restructures to problem-first flow. Opens with the prospect's pain point, bridges to the solution, closes with a single low-friction CTA. Moves social proof from the middle (where it interrupts) to right before the CTA (where it reduces friction).

**The Sharpener:** Replaces "We help companies improve their workflow" with "We cut onboarding time from 3 weeks to 3 days for teams like [specific company]." Replaces "reach out" with "reply." Eliminates every instance of passive voice.

**The Humanizer:** Adds a personalization slot for the prospect's recent work. Changes the tone from "vendor pitch" to "peer sharing something useful." Adds a PS line with a relevant insight (not a sales push) that gives value even if they never reply.

**The Stress-Tester:** Tests the email against a skeptical reader. Identifies that the CTA assumes interest ("When can we chat?") — replaces with a no-commitment offer ("Here's a 2-min case study if useful"). Adds a one-line opt-out that prevents spam flags.

**Master Forger's Synthesis:**

*The Forged Version:* A 95-word email that opens with the prospect's specific pain (Architect's structure + Humanizer's personalization), uses one precise proof point (Sharpener), includes a no-commitment CTA (Stress-Tester), and has zero filler (Surgeon). The PS line stays (Humanizer) — peer reviewers unanimously flagged it as the single best addition.

*What Changed:* 53% shorter. Problem-first structure. Specific proof replacing generic claims. CTA that doesn't presume interest.

*What the Forge Couldn't Fix:* The prospect-specific personalization slot needs the user to fill in actual details for each send. The proof point needs a real metric from a real client.
