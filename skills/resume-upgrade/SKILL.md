---
name: resume-upgrade
description: Propose how to upgrade a resume by identifying missing projects, skills, evidence, and experience-building moves, but only after a `resume-critic` review has been completed for a specific job description or job title. Use when the user wants strategic suggestions beyond the current resume.
---

# Resume Upgrade

A resume expansion workflow that starts from `resume-critic`, then identifies what the candidate is missing for the target job and proposes honest ways to build or demonstrate that missing signal over time.

## When to Use

Use this skill when the user wants strategic upgrades beyond simple rewriting: new project ideas, evidence gaps, missing skill development, portfolio improvements, or stronger credibility signals for a target role.

**Good uses:**
- "What should I build to become a stronger candidate for this ML engineer role?"
- "Based on the critique, what projects or skills should I add over the next 2 months?"
- "How do I upgrade this resume so it becomes more competitive for data science jobs?"

**Bad uses:**
- "Rewrite my existing bullets only." (use `resume-rewrite`)
- "Review my current resume." (use `resume-critic`)
- "Suggest random good projects with no job target." (insufficient targeting)
- "Pretend I already did these projects and add them to my resume." (dishonest fabrication)

This skill cannot start without a prior `resume-critic` output for the same resume and target job.

## Hard Dependency

Before upgrading, require all three:
1. The current resume text or file
2. The target job description, or at minimum the concrete job title
3. The output from `resume-critic` for that same target job

If the user has not run `resume-critic`, stop and ask for it first.
If the critique exists but targets a different job, do not reuse it.

## What "Upgrade" Means

An upgrade proposes new signal the user could honestly create, learn, or document, such as:
- New portfolio projects
- Missing skills to learn
- Missing certifications or coursework worth considering
- Better evidence collection from existing work
- Open-source, freelance, volunteer, or side-project moves that strengthen credibility

An upgrade does not mean fabricating content into the resume immediately.

## Workflow

### Step 1: Gather Inputs

Read:
- The current resume
- The target job description or job title
- The `resume-critic` output
- Any user constraints such as time, domain, experience level, or timeline

Start the framed brief with:
- `This upgrade plan is for: [full job title]`
- `Job description source: [pasted JD | linked JD | inferred from user-provided title only]`
- `Critique source: [resume-critic transcript/report reference or pasted output]`

### Step 2: Identify Signal Gaps

Use the `resume-critic` findings plus the job target to identify:
1. Missing hard skills
2. Missing domain evidence
3. Missing project proof
4. Missing ownership or impact signals
5. Missing portfolio or public proof points

Separate:
- gaps that can be solved by documenting existing work better
- gaps that require genuinely new work

### Step 3: Produce The Upgrade Plan

Recommend concrete upgrades such as:
- 2-5 project ideas tailored to the target job
- Skills to learn, with rationale
- Evidence to gather from existing work
- Optional portfolio, GitHub, case study, or certification moves

Every suggestion must be justified against the stated target job.

### Step 4: Prioritize By ROI

Rank upgrades by:
- speed to complete
- signal strength for the target job
- realism given the user's likely starting point

## Output Format

Use this exact structure:

```text
## Job Target
This upgrade plan is for: [job title]
Job description source: [source]

## Resume-Critic Basis
[Short summary of the critique being acted on]

## What The Resume Already Has
[Existing strengths worth preserving]

## Gaps To Close
[The highest-value missing signals]

## Upgrade Plan
[Concrete projects, skills, and proof-building moves]

## Fastest High-ROI Wins
[The quickest upgrades with the best payoff]

## What Not To Fake
[Short warning about claims that must only be added after real work is done]
```

## Important Rules

- Do not start without `resume-critic`.
- Do not start without a target job description or at least a concrete job title.
- Always state clearly which job the upgrade plan is for.
- Do not present suggested projects or skills as if they already belong on the resume.
- Keep suggestions realistic, job-linked, and honest.
- Be honest about the size of gaps. If the user is far from the target role, say so directly. A politely vague upgrade plan wastes their time.

