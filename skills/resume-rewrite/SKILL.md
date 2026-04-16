---
name: resume-rewrite
description: Rewrite a resume using only the candidate's existing material, but only after a `resume-critic` review has been completed for a specific job description or job title. Use when the user wants the resume rewritten, tightened, or retargeted without inventing new experiences.
---

# Resume Rewrite

A resume rewriting workflow that converts `resume-critic` findings into a sharper, targeted resume using only the material already present in the resume and directly supported supporting notes.

## When to Use

Use this skill when the user wants the current resume rewritten for a specific job, but does not want invented experiences, fake metrics, or speculative new projects.

**Good uses:**
- "Rewrite this resume for this Senior Data Analyst job."
- "Use the critique and tighten my bullets for this PM role."
- "Retarget my current resume for this backend engineer position."

**Bad uses:**
- "Invent better projects for me." (use `resume-upgrade`)
- "Suggest skills I should learn and add later." (use `resume-upgrade`)
- "Rewrite this with no prior critique." (missing required input)
- "Make me look more senior even if the resume does not support it." (fabrication task)

This skill cannot start without a prior `resume-critic` output for the same resume and target job.

## Hard Dependency

Before rewriting, require all three:
1. The current resume text or file
2. The target job description, or at minimum the concrete job title
3. The output from `resume-critic` for that same target job

If the user has not run `resume-critic`, stop and ask for it first.
If the critique exists but targets a different job, do not reuse it. Run `resume-critic` again for the correct target first.

## Core Rule

Rewrite only from existing material:
- Existing resume bullets
- User-provided supporting notes
- User-provided portfolio/project descriptions
- Facts already present in the workspace

Do not invent:
- New projects
- New skills not already supported
- Fake metrics
- Unsupported ownership claims
- Degrees, certifications, or tools the user never mentioned

## Workflow

### Step 1: Gather Inputs

Read:
- The current resume
- The target job description or job title
- The `resume-critic` output
- Any user-provided factual notes that can support stronger phrasing

Start the framed brief with:
- `This rewrite is for: [full job title]`
- `Job description source: [pasted JD | linked JD | inferred from user-provided title only]`
- `Critique source: [resume-critic transcript/report reference or pasted output]`

### Step 2: Extract Rewrite Priorities

From the `resume-critic` output, identify:
1. The top 5 fixes
2. The one thing to fix first
3. Target keywords and concepts that are missing or underemphasized
4. Bullets or sections that are actively hurting interview conversion

### Step 3: Rewrite The Resume

Produce an updated version of the resume that:
- Improves targeting for the stated job
- Tightens weak bullets
- Surfaces stronger evidence earlier
- Removes low-signal filler
- Preserves factual truth

If there is not enough factual support to strengthen a claim honestly, leave it weaker rather than fabricating detail.

### Step 4: Explain The Changes

After the rewritten resume, provide:
- A short summary of the main changes
- A mapping from the top `resume-critic` findings to the rewrite decisions
- A short list of facts still missing from the resume that blocked stronger rewriting

## Output Format

Use this exact structure:

```text
## Job Target
This rewrite is for: [job title]
Job description source: [source]

## Resume-Critic Basis
[Short summary of the critique being acted on]

## Rewritten Resume
[Full rewritten resume]

## What Changed
[Concise explanation of the biggest rewrite choices]

## Missing Facts That Limited The Rewrite
[Only facts the user would need to provide for a stronger version]
```

## Important Rules

- Do not start without `resume-critic`.
- Do not rewrite against an unspecified target job.
- Do not invent accomplishments.
- Do not add new projects or speculative skills. That belongs in `resume-upgrade`.
- Keep the rewritten resume visibly aligned to the target job named at the top.

