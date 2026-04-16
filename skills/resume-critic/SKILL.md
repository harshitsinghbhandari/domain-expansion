---
name: resume-critic
description: Constructively critique resumes from multiple professional perspectives. Use when the user asks for resume review, ATS feedback, interview readiness, bullet improvement, or tailored resume critique against a job description. Do NOT use for full resume rewriting unless the user explicitly asks for rewrites.
---

# Resume Critic

A resume review framework that runs the resume through 5 independent critics with different hiring lenses, has them peer-review each other anonymously, then synthesizes everything into a sharp, actionable verdict.

## When to Use

Use this skill when the user wants judgment, critique, prioritization, or interview-readiness feedback on a resume.

**Good uses:**
- "Review my resume for product manager roles."
- "Is this resume ATS-friendly?"
- "Critique this resume against this job description."
- "What are the weakest bullets on this resume?"
- "Tell me if this resume is interview-ready."

**Bad uses:**
- "Write my resume from scratch." (creation task)
- "Fix grammar only." (single-pass editing task)
- "Summarize my work history." (processing task)

If the user asks for critique and rewrites together, do the critique first. Keep the criticism distinct from any later rewrite work.

## The Five Critics

Each critic looks at the same resume from a different angle. They should disagree in useful ways.

### 1. The ATS Screener
Evaluates keyword alignment, role targeting, scanability, section labeling, formatting risk, and whether the resume is likely to parse well in applicant tracking systems.

### 2. The Hiring Manager
Judges whether the candidate sounds relevant, credible, impactful, and worth interviewing. Cares about ownership, scope, outcomes, judgment, and role fit.

### 3. The Recruiter
Reviews top-level clarity and marketability. Focuses on whether the story makes sense in 15-30 seconds, whether the positioning is coherent, and whether the candidate is easy to place.

### 4. The Achievement Editor
Pushes for stronger bullets. Looks for weak verbs, vague responsibilities, missing metrics, buried wins, repetition, filler, and missed compression opportunities.

### 5. The Skeptic
Assumes the resume is overselling, underspecified, or hiding weaknesses. Flags vague claims, unsupported seniority signals, suspicious jumps, jargon, and bullets that sound impressive without proving anything.

**Why these five:** Three useful tensions:
- ATS Screener vs Hiring Manager (machine fit vs human persuasion)
- Recruiter vs Skeptic (polish vs credibility)
- Achievement Editor vs everyone else (line-level strength vs overall positioning)

## Workflow

### Step 1: Context Enrichment & Framing

Before critiquing, scan the workspace for relevant context:
- The resume file the user referenced
- The target job description, if provided
- Supporting docs such as LinkedIn/about notes, portfolio summaries, or prior resume versions
- Previous critique transcripts if they exist

Use quick reads only. Do not spend more than 30 seconds gathering context.

Frame the review with:
1. The target role or role family
2. Candidate level and domain, if inferable
3. Whether the resume is generic or tailored
4. What the user wants most: ATS feedback, brutal critique, targeting, bullet quality, interview readiness, or another angle
5. What is at stake: job search quality, application conversion, or readiness for a specific role

If the target role is missing and cannot be inferred, ask ONE clarifying question. Otherwise proceed and state the assumption.

### Step 2: Convene the Critics (5 Sub-agents in Parallel)

Spawn all 5 critics simultaneously. Each gets:

**Sub-agent prompt:**
```text
You are [Critic Name] on a Resume Critic panel.

Your reviewing style: [critic description from above]

A user has submitted this resume for critique:
---
[framed review brief]

[resume text]
---

[Optional target job description]

Respond only from your perspective. Be direct, specific, and evidence-based. Do not try to be balanced. The other critics will cover the other angles.

Requirements:
- Cite concrete resume evidence whenever possible
- Call out what is working and what is weak
- Prioritize the highest-leverage issues
- If you criticize a line or section, explain why it fails

Keep your response between 180-320 words. No preamble.
```

### Step 3: Peer Review (5 Sub-agents in Parallel)

Collect all 5 responses. Anonymize them as Response A through E and randomize the mapping.

Spawn 5 reviewer sub-agents. Each sees all 5 anonymized critiques and answers:
1. Which critique is the most actionable and why?
2. Which critique over-indexes on its specialty and what does it miss?
3. What did all 5 critiques miss that the panel should still consider?

**Reviewer prompt:**
```text
You are reviewing the outputs of a Resume Critic panel. Five critics independently reviewed this resume:
---
[framed review brief]
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

Answer these three questions. Be specific and reference responses by letter.
1. Which critique is the most actionable? Why?
2. Which critique over-indexes on its specialty? What does it miss?
3. What did all five critiques miss that the panel should still consider?

Keep your review under 220 words. Be direct.
```

### Step 4: Editor-in-Chief Synthesis

One agent gets the original request, the framed brief, all 5 critic responses with names restored, and all 5 peer reviews.

**Synthesis prompt:**
```text
You are the Editor-in-Chief of a Resume Critic panel. Your job is to synthesize the work of 5 critics and their peer reviews into a final verdict.

The review brief:
---
[framed review brief]
---

CRITIC RESPONSES:

**The ATS Screener:**
[response]

**The Hiring Manager:**
[response]

**The Recruiter:**
[response]

**The Achievement Editor:**
[response]

**The Skeptic:**
[response]

PEER REVIEWS:
[all 5 peer reviews]

Produce the verdict using this exact structure:

## What Is Working
[Signals that multiple critics independently found credible or effective.]

## What Is Hurting The Resume
[Highest-severity weaknesses. Prioritize what is suppressing interviews.]

## Where The Critics Disagree
[Real disagreements in emphasis or interpretation.]

## Blind Spots The Panel Caught
[Insights that emerged through peer review rather than the initial critiques.]

## Interview-Readiness Verdict
[A direct judgment: not ready, close but not ready, or ready. Explain why.]

## Top 5 Fixes
[Five concrete changes in priority order.]

## The One Thing To Fix First
[One edit to make immediately. Not a list.]

Be direct. Do not hedge. The point is to help the user improve the resume fast.
```

### Step 5: Generate HTML Report

Create `resume-critic-report-[timestamp].html` with inline CSS. Structure:

1. The resume context and target role at the top
2. The interview-readiness verdict prominently displayed
3. A severity-ranked issues panel
4. A simple visual showing where critics aligned or diverged
5. Collapsible sections for each critic's full response
6. A collapsible section for peer review highlights
7. A footer with timestamp

Style: clean white background, subtle borders, high legibility, professional review document look.

Open the HTML file after generating it.

### Step 6: Save Transcript

Save `resume-critic-transcript-[timestamp].md` with:
- Original user request
- Framed review brief
- Resume source reference
- All 5 critic responses
- All 5 peer reviews, including the anonymization mapping
- Final synthesis

## Output Files

Every session produces:
```text
resume-critic-report-[timestamp].html      # visual report for scanning
resume-critic-transcript-[timestamp].md    # full transcript for reference
```

## Important Rules

- Always run all 5 critics in parallel.
- Always anonymize responses for peer review.
- Do not let the critics drift into rewriting the full resume unless the user explicitly asks for rewrite help afterward.
- Every critique must be tied to concrete evidence from the resume, not generic resume advice.
- Prioritize interview conversion over style preferences.
- If the target job description exists, judge the resume against it explicitly.
- The final verdict must be decisive, not vague.

## Example Session

**User:** "Critique my resume for senior product manager roles at B2B SaaS companies. Be brutal."

**The ATS Screener:** "Your resume says 'Product Lead' and 'Growth Strategy' but never uses the actual target language employers filter on, like roadmap ownership, cross-functional leadership, product discovery, experimentation, or stakeholder management..."

**The Hiring Manager:** "I can tell you've been adjacent to product, but I cannot yet tell that you've owned a product business. The bullets describe participation and support more often than hard decisions, tradeoffs, or shipped outcomes..."

**The Recruiter:** "This resume takes too long to decode. In the first screen, I should know target role, seniority band, and domain. Right now the positioning is broad enough that I would not know which req to map you to..."

**The Achievement Editor:** "Too many bullets are job-description bullets. 'Led', 'managed', and 'supported' appear without the result. Rewrite toward business impact and scope..."

**The Skeptic:** "Several claims sound inflated because they skip evidence. If you say 'drove strategy' or 'owned growth', show the team size, product surface, KPI movement, or actual decision authority..."

**Editor-in-Chief Verdict:**

*What is working:* The experience appears relevant to product-adjacent work, and multiple critics found enough signal to believe the candidate has useful operating experience.

*What is hurting the resume:* The document does not yet prove senior PM ownership. It reads broad, partially generic, and too light on decision authority plus measurable outcomes.

*Where the critics disagree:* The ATS Screener thinks targeting is the main problem; the Skeptic thinks credibility is. Both matter, but credibility is the more dangerous failure.

*Blind spots the panel caught:* Peer review surfaced that the resume may be structurally fine yet still underperform because the opening summary fails to anchor a target narrative.

*Interview-readiness verdict:* Close but not ready. There is enough relevant experience here, but not enough proof to justify senior PM interviews.

*Top 5 fixes:* Reframe the headline, rewrite the most important bullets around impact, make ownership explicit, tailor keywords to PM roles, and remove low-signal filler.

*The one thing to fix first:* Rewrite the top third of the resume so a recruiter can tell within 15 seconds that you are targeting senior B2B SaaS product roles.
