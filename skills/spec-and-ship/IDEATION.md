# Ideation: Interview-Driven Development

This is the first half of spec-and-ship: the pre-AO work that produces the spec. Once it's done, move on to [EXECUTION.md](EXECUTION.md).

## The idea in one paragraph

Most people automate the wrong end of the loop. They race to automate the *building*: generate code faster, spin up more agents, parallelize. But building was rarely the real bottleneck. The expensive failure mode isn't slow construction; it's confidently building the *wrong thing* fast, because nobody pinned down what it should actually do. The real bottleneck is **under-specified intent**, and no amount of execution speed fixes vague. So invert the loop: before any code gets written, the machine interrogates the *human* and extracts a complete, opinionated spec. Automate the execution. Never automate the deciding.

This half has a gate, then four phases.

## The gate: is this even worth it?

Interview-driven development is deliberate overhead. You spend real thinking time and a round of Q&A *before* a line of code exists. That cost only pays off when **building the wrong thing would be expensive**: a sizeable feature, a new subsystem, a schema change, a public API, a migration, a refactor that touches many files, anything hard to walk back.

For a bug fix, a one-liner, a cosmetic tweak, or anything you'd finish faster than you could interview about, **skip it and just do the work.** Forcing this ritual onto small tasks is its own anti-pattern. When in genuine doubt on something medium-sized, ask the user: "This looks substantial enough that it's worth pinning the spec down first. Want me to interview you, or should I just start?"

If it clears the gate, continue.

## Phase 1: Get the raw intent out of the human's head

Before you ask anything, the user needs their unfiltered intent on the page. Don't let them hand you two vague sentences and expect you to interview greatness out of nothing; you'll just produce plausible-sounding mush.

Invite a brain-dump. Ask the user to tell you, in their own words and without worrying about structure: what the feature does, why it exists, who it's for, what parts of the system it touches, and, most important, **everything they're unsure about or haven't decided.** Encourage messy and incomplete over tidy and thin. If they already have a spec doc, PRD, or design brief, that *is* the raw intent: read it closely and treat it as Phase 1 input.

The goal of this phase is raw material, not polish. You can't extract good decisions from someone who hasn't yet said what they actually want.

## Phase 2: Interview them (this is the whole skill)

Now read what they gave you and **interrogate it.** Generate a batch of pointed questions, typically **20 to 30 for a real feature**, that surface every decision the user left implicit, vague, or unmade. This is the highest-leverage step; the quality of your questions *is* the value. Go deep. Think like the engineer who'll be three days into the build hitting each ambiguity in turn, and ask it now, while it costs a sentence instead of a torn-down branch.

**Cover the full surface.** Don't just ask about the happy path. Pull questions from across these axes (not every axis applies to every feature; use judgment):

- **Product / UX:** What exactly does the user see and do? What's the actual flow, states, empty states, loading, success, the copy that matters?
- **Scope boundaries:** What is explicitly *out* of scope? This is the most skipped and most valuable category. Nail down what you're *not* building.
- **Edge cases and failure modes:** Empty inputs, huge inputs, concurrent actions, partial failures, network loss, the thing that happens 1% of the time.
- **Data and state:** Shape of the data, source of truth, what persists, what's derived, validation rules, what happens to existing/old data.
- **Design / architecture:** Where does this live, what does it depend on, what's the boundary, what's the interface, sync vs async, where's the seam.
- **Error handling:** What does the user see when it breaks? Retry, fail loud, fail silent, fall back?
- **Non-functional:** Performance expectations, scale, security, auth, permissions, accessibility, observability, *only where they genuinely matter.*
- **Integration and dependencies:** What does this talk to? What contracts must it honor? What breaks downstream if this changes?
- **Rollout and migration:** Feature-flagged? Backfill? Reversible? How does it ship?

**Make the questions cheap to answer.** A wall of 30 open-ended questions is exhausting and gets half-answered. Instead:

- **Group by theme** so the user can rip through related decisions together.
- **Offer an opinionated default with each question** wherever you can: your recommended answer plus a one-line why. This models good judgment, lowers the cost to a thumbs-up, and gives the user something concrete to push against. "For empty state I'd show a first-run prompt rather than a blank list, agree?" beats "What should the empty state be?"
- **Don't ask what they already answered.** Mining the Phase 1 material for questions it already resolves wastes the user's trust and time. Ask only the real open forks.
- **Flag the load-bearing ones.** If three of the thirty questions actually determine the architecture, say so, so the user spends their attention there.

Present the questions, then let the user answer in their own words. They should answer **opinionatedly**: this is them deciding, not brainstorming. If an answer opens a new fork, ask the follow-up. Keep going until the ambiguity is genuinely drained, not until you hit a question count.

## Phase 3: Synthesize the resolved spec

Once the questions are answered, roll **their answers** into a single, complete, self-contained spec. This is the artifact the whole process exists to produce: the user's decisions, made explicit and durable. Their answers *are* the spec.

Write it in your own words, fully resolved: no open questions left dangling, no "TBD". Use roughly this structure (adapt to fit):

```
# [Feature name]

## Goal
One paragraph: what this does and why it exists.

## Scope
In scope: ...
Explicitly out of scope: ...

## Behavior
The resolved decisions, walked through: flows, states, data, edge cases,
error handling, everything settled in the interview, stated plainly.

## Design / approach
Where it lives, what it depends on, the key interfaces and seams.

## Open decisions deferred to build time
(Only genuinely small calls. If something big is still open, you're not done
interviewing: go back to Phase 2.)

## Task breakdown
The work, split into independently-shippable pieces, grouped into batches
where each batch is a set of tasks that can be built in parallel.
```

The **task breakdown** is what Execution consumes: independently-shippable pieces grouped into dependency-ordered batches. Spend care here; it's the seam between the two halves.

Then confirm it back: "Here's the resolved spec, does this match what you decided?" The user should recognize their own intent, sharpened. If they don't, the interview missed something; fix it before any building starts.

## Phase 4: Hand off to execution

Now, and only now, building begins. The resolved spec, especially its batched task breakdown, is the input to [EXECUTION.md](EXECUTION.md), where the AO orchestrator spawns workers batch by batch.

Autonomy is not abdication. **The one rule for build time:** if whoever's building hits a product or design decision the spec doesn't cover, they **stop and ask**; they don't guess, and they don't stall silently. A surfaced fork costs one question and one answer. A guessed-wrong fork costs a rebuild. This is the difference between being *out of the loop* and being *in the loop at the points that need a human.* In Execution, this rule is enforced mechanically: workers escalate forks via `inform` (see TEMPLATE.md).

## What this method can't do

Be honest about the limits; don't oversell it.

- **It surfaces ambiguity; it doesn't supply judgment.** If the user's answers to the interview are wrong, they've now built the wrong thing *efficiently.* Garbage in, faster out. The method makes intent explicit; it can't make intent good. That part stays with the human, which is exactly why the human stays in the loop at the forks.
- **It's overhead by design.** On small work it's pure cost. Respect the gate.
- **The questions are the product.** A lazy, shallow interview produces a lazy spec. The 20 to 30 questions earn their place only if they're sharp enough to catch the things the user would otherwise have tripped over. Spend your effort there.

## The one line to remember

Automate the execution. Never automate the deciding.
