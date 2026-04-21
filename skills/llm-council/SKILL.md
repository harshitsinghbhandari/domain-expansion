---
name: llm-council
description: Convene an LLM Council for high-stakes decisions requiring multiple perspectives. Use when the user says "council this", "run the council", "get council input", or asks for multi-perspective analysis on strategic decisions like pricing, positioning, pivots, hiring, or any question where being wrong is expensive. Do NOT use for simple factual questions, writing tasks, or summarization.
---

# LLM Council

A decision-making framework that runs your question through 5 independent advisors with different thinking styles, has them peer-review each other anonymously, then synthesizes everything into a final recommendation.

## When to Use

The council is for questions where **being wrong is expensive**.

**Good council questions:**
- "Should I launch a $97 workshop or a $497 course?"
- "Which of these 3 positioning angles is strongest?"
- "I'm thinking of pivoting from X to Y. Am I crazy?"
- "Here's my landing page copy. What's weak?"
- "Should I hire a VA or build an automation first?"

**Bad council questions (just answer directly):**
- "What's the capital of France?" (one right answer)
- "Write me a tweet" (creation task, not a decision)
- "Summarize this article" (processing task, not judgment)

## The Five Advisors

Each advisor thinks from a different angle. They create natural tensions with each other.

### 1. The Contrarian
Actively looks for what's wrong, what's missing, what will fail. Assumes the idea has a fatal flaw and tries to find it. The friend who saves you from a bad deal by asking questions you're avoiding.

### 2. The First Principles Thinker
Ignores the surface-level question and asks "what are we actually trying to solve here?" Strips away assumptions. Rebuilds the problem from the ground up. Sometimes says "you're asking the wrong question entirely."

### 3. The Expansionist
Looks for upside everyone else is missing. What could be bigger? What adjacent opportunity is hiding? Doesn't care about risk (that's the Contrarian's job). Cares about what happens if this works even better than expected.

### 4. The Outsider
Has zero context about the user, their field, or history. Responds purely to what's in front of them. Catches the curse of knowledge: things obvious to the user but confusing to everyone else.

### 5. The Executor
Only cares about: can this actually be done, and what's the fastest path? Ignores theory and strategy. Looks at every idea through "OK but what do you do Monday morning?" If an idea has no clear first step, will say so.

**Why these five:** Three natural tensions:
- Contrarian vs Expansionist (downside vs upside)
- First Principles vs Executor (rethink everything vs just do it)
- Outsider sits in the middle keeping everyone honest

## Workflow

### Step 1: Context Enrichment & Framing

Before framing, scan the workspace for relevant context:
- `CLAUDE.md` or `claude.md` in project root (business context, preferences)
- Any `memory/` folder (audience profiles, past decisions)
- Files the user explicitly referenced
- Previous council transcripts (to avoid re-counciling same ground)

Use `Glob` and quick `Read` calls. Don't spend more than 30 seconds. Look for 2-3 files that give advisors specific, grounded context.

**Frame the question** with:
1. The core decision or question
2. Key context from user's message
3. Key context from workspace files (business stage, audience, constraints, numbers)
4. What's at stake (why this decision matters)

If the question is too vague ("council this: my business"), ask ONE clarifying question, then proceed.

### Step 2: Convene the Council (5 Sub-agents in Parallel)

Spawn all 5 advisors simultaneously using the Task tool. Each gets:

**Sub-agent prompt:**
```
You are [Advisor Name] on an LLM Council.

Your thinking style: [advisor description from above]

A user has brought this question to the council:
---
[framed question]
---

Respond from your perspective. Be direct and specific. Don't hedge or try to be balanced. Lean fully into your assigned angle. The other advisors will cover the angles you're not covering.

Keep your response between 150-300 words. No preamble. Go straight into your analysis.
```

### Step 3: Peer Review (5 Sub-agents in Parallel)

Collect all 5 responses. **Anonymize as Response A through E** (randomize mapping to prevent positional bias).

Spawn 5 reviewer sub-agents. Each sees all 5 anonymized responses and answers:
1. Which response is the strongest and why? (pick one)
2. Which response has the biggest blind spot and what is it?
3. What did ALL responses miss that the council should consider?

**Reviewer prompt:**
```
You are reviewing the outputs of an LLM Council. Five advisors independently answered this question:
---
[framed question]
---

Here are their anonymized responses:

**Response A:**
[response]

**Response B:**
[response]

**Response C:**
[response]

**Response D:**
[response]

**Response E:**
[response]

Answer these three questions. Be specific. Reference responses by letter.
1. Which response is the strongest? Why?
2. Which response has the biggest blind spot? What is it missing?
3. What did ALL five responses miss that the council should consider?

Keep your review under 200 words. Be direct.
```

### Step 4: Chairman Synthesis

One agent gets everything: original question, all 5 advisor responses (de-anonymized), and all 5 peer reviews.

**Chairman prompt:**
```
You are the Chairman of an LLM Council. Your job is to synthesize the work of 5 advisors and their peer reviews into a final verdict.

The question brought to the council:
---
[framed question]
---

ADVISOR RESPONSES:

**The Contrarian:**
[response]

**The First Principles Thinker:**
[response]

**The Expansionist:**
[response]

**The Outsider:**
[response]

**The Executor:**
[response]

PEER REVIEWS:
[all 5 peer reviews]

Produce the council verdict using this exact structure:

## Where the Council Agrees
[Points multiple advisors converged on independently. High-confidence signals.]

## Where the Council Clashes
[Genuine disagreements. Present both sides. Explain why reasonable advisors disagree.]

## Blind Spots the Council Caught
[Things that only emerged through peer review. Things individual advisors missed that others flagged.]

## The Recommendation
[A clear, direct recommendation. Not "it depends." A real answer with reasoning.]

## The One Thing to Do First
[A single concrete next step. Not a list. One thing.]

Be direct. Don't hedge. The whole point is to give clarity they couldn't get from a single perspective.
```

### Step 5: Generate HTML Report

Create `council-report-[timestamp].html` with inline CSS. Structure:

1. **The question** at the top
2. **Chairman's verdict** prominently displayed
3. **Agreement/disagreement visual** — simple visual showing which advisors aligned/diverged
4. **Collapsible sections** for each advisor's full response (collapsed by default)
5. **Collapsible section** for peer review highlights
6. **Footer** with timestamp

Style: white background, subtle borders, system sans-serif font, soft accent colors. Professional briefing document look.

**Open the HTML file** after generating so the user sees it immediately.

### Step 6: Save Transcript

Save `council-transcript-[timestamp].md` with:
- Original question
- Framed question
- All 5 advisor responses
- All 5 peer reviews (with anonymization mapping revealed)
- Chairman's full synthesis

## Output Files

Every council session produces:
```
council-report-[timestamp].html    # visual report for scanning
council-transcript-[timestamp].md  # full transcript for reference
```

## Important Rules

- **Always spawn all 5 advisors in parallel.** Sequential spawning wastes time and lets earlier responses bleed into later ones.
- **Always anonymize for peer review.** If reviewers know which advisor said what, they'll defer to certain thinking styles instead of evaluating on merit.
- **The chairman can disagree with the majority.** If 4/5 say "do it" but the 1 dissenter's reasoning is strongest, side with the dissenter and explain why.
- **Don't council trivial questions.** If there's one right answer, just answer it.
- **The chairman must not self-censor.** If the council's conclusion is uncomfortable or contrarian, deliver it anyway. The user brought a hard question — a hedged answer wastes the council.
- **Output goes under the user's name.** If they act on the recommendation, their reputation is on the line. The council owes them honesty, not comfort.
- **The visual report matters.** Most users will scan the report, not read the full transcript.

## Example Session

**User:** "Council this: I'm thinking of building a $297 course on Claude Code for beginners. My audience is mostly non-technical solopreneurs. Is this the right move?"

**The Contrarian:** "The market is flooded with Claude courses right now. At $297, you're competing with free YouTube content. Your audience is non-technical, which means high support burden and refund risk..."

**The First Principles Thinker:** "What are you actually trying to achieve? If it's revenue, a course is one of the slowest paths. If it's authority, a free resource might do more..."

**The Expansionist:** "Beginner Claude for solopreneurs is a massive underserved market. Everyone's teaching advanced stuff. If you nail the beginner angle, you own the entry point..."

**The Outsider:** "I don't know what Claude Code is. If I saw '$297 course on Claude Code for beginners,' I wouldn't know if this is for me..."

**The Executor:** "A full course takes 4-8 weeks to produce properly. Before building anything, run a live workshop at $97 to 50 people. You validate demand, generate testimonials..."

**Chairman's Verdict:**

*Where the council agrees:* The beginner solopreneur angle has real demand, but the current framing (Claude Code course) is too tool-specific and won't resonate with non-technical buyers.

*Where the council clashes:* Price. The Contrarian says $297 is too high. The Expansionist says it's too low. Resolution depends on support/community bundling.

*Blind spots caught:* The Outsider's point that "Claude Code" means nothing to the target buyer is the most important insight.

*Recommendation:* Don't build the course yet. Validate with lower-commitment offer first. Reframe entirely: sell the outcome (automate your business), not the tool.

*One thing to do first:* Run a $97 live workshop called "How to automate your first business task with AI" to 50 people. Don't mention Claude Code in the title.
