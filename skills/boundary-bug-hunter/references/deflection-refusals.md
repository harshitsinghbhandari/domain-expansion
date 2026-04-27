# Deflection Refusals — Verbatim Responses

The six named deflections from the originating issue. When the skill encounters one — in PR descriptions, code comments, the user's reply, or its own draft reasoning — it identifies the deflection by number and refuses with the corresponding response.

The refusal is the point. Surfacing the deflection is half the value; absorbing it would be the failure mode.

When using these in a report, attribute the source ("PR description, line 12", "author comment on this skill's last run") and quote it. Then deliver the refusal text.

---

## Deflection 1: "It's a unit-test responsibility."

> **Refused.** Boundary bugs live *between* units, by definition. Each unit's tests pass in isolation — that's the failure mode this skill is built to catch, not the defense against it. A unit-test responsibility argument applies to bugs that fit inside a unit. The boundary at `<producer>:<line>` → `<consumer>:<line>` does not. Add the round-trip test, or record an explicit out-of-scope reason in `.boundary-deferrals.json`.

**Where it appears:** Author replies to review comments on the seam. Reviewer notes asking "why isn't this covered" answered with "covered at the unit level".

**Why it's tempting:** Unit tests are easier to write and run faster. The author has already spent that effort and reasonably wants credit.

**Why it fails:** The seam is the bug. Unit tests by construction don't run the seam.

---

## Deflection 2: "The contract is documented."

> **Refused.** A documented contract that is never executed is a hope, not a check. Documentation drifts; tests do not. If the producer at `<producer>:<line>` and the consumer at `<consumer>:<line>` agree on a contract, prove it with a test that runs both. If they disagree, the documentation just lied. Either way, the round-trip test is the only artifact that surfaces the truth. Add it, or defer with a written reason.

**Where it appears:** Comments above functions describing return shape; OpenAPI specs; type definitions; design docs.

**Why it's tempting:** The author did write documentation. The contract is, in some sense, "specified".

**Why it fails:** Documentation captures intent. Tests capture reality. The bug we're hunting is the gap between the two.

---

## Deflection 3: "It works in isolation."

> **Refused.** Isolation is the failure mode under investigation. Every unit on every side of every boundary bug works in isolation — that's why they ship. The whole purpose of this check is to find what breaks when the units are composed. Add a test that composes them at `<producer>:<line>` → `<consumer>:<line>`. If you cannot, record why in `.boundary-deferrals.json`.

**Where it appears:** Author comments saying "I tested this locally", "the function returns the right thing", "the consumer handles this case".

**Why it's tempting:** The author *did* verify each side. They feel they did the work.

**Why it fails:** Verifying both sides separately is what the skill is rejecting. The defense and the failure mode are the same artifact.

---

## Deflection 4: "We'd catch it in manual / staging / production."

> **Refused.** Catching a boundary bug in manual QA, staging, or production means: real users or operators experience the symptom, an incident is opened, the team backtracks to the seam, and the seam test gets written *after* the cost has been paid. This skill exists to write that test *before*. The boundary at `<producer>:<line>` → `<consumer>:<line>` is unverified in CI, which is the only environment that gates merges. Add the test, or defer.

**Where it appears:** "QA will hit this", "we'll see it in canary", "Sentry will tell us", "we have observability".

**Why it's tempting:** The org does have downstream environments. Production observability is real.

**Why it fails:** Downstream environments find symptoms after the cost. CI finds bugs before. The skill's whole value is shifting boundary bugs left.

---

## Deflection 5: "The reviewer will catch it."

> **Refused.** Reviewers also read units in isolation. The pattern this skill is built around — every side passes its own tests, the seam is broken, the bug ships anyway — is exactly the pattern reviewers consistently miss. Outsourcing the verification to a future human read is what produced the boundary bugs in the first place. The reviewer can verify the test exists; they cannot reliably verify the seam holds without the test running. Add the test, or defer.

**Where it appears:** "I'll let the reviewer catch this", "marking ready for review and asking", "open question for reviewer".

**Why it's tempting:** Reviewers do catch many bugs. The author is being humble.

**Why it fails:** Reviewers catch obvious bugs. Boundary bugs are catch-resistant by their nature — they require running the seam, which a reviewer typically does not.

---

## Deflection 6: "It's a small change."

> **Refused.** Size is uncorrelated with blast radius at boundaries. A one-line change to a producer's return shape can break every consumer downstream silently. A one-line change to a consumer's parse logic can corrupt every persisted record going forward. The size of the diff at `<producer>:<line>` → `<consumer>:<line>` is not the variable that determines whether a round-trip test is needed; the existence of the seam is. Add the test, or defer.

**Where it appears:** "It's just a rename", "tiny tweak", "one-character change", "drive-by fix".

**Why it's tempting:** Small changes feel low-risk. The author wants to ship and move on.

**Why it fails:** Small at the source ≠ small downstream. The whole point of seams is that effects propagate.

---

## Composite deflections

If the author replies with a stack ("it's small AND documented AND we'll catch it in staging"), refuse each component. Don't let the stack count as one objection.

If the author invents a new deflection not on the list, document it in the report under "Novel deflections seen this run" and refuse on first principles: the skill exists because seam bugs ship, the round-trip test is the only fix that has been shown to prevent them, and any reasoning that ends in "skip the test" is the failure mode.

---

## What is *not* a deflection

These are legitimate reasons to defer (record in `.boundary-deferrals.json`, do not refuse):

- "The consumer is a third-party service we don't control and can't run in CI." → Defer, document the contract assumption, and request a contract test against a sandbox if available.
- "The boundary requires production-only infrastructure (e.g., AWS-managed identity) that we don't have in CI." → Defer, document, and add to a backlog for a long-running integration environment.
- "The boundary is between the application and a managed cloud primitive whose contract is owned by the cloud provider." → Defer, document, but verify the consumer handles the cloud's documented failure modes (timeouts, throttling, eventual consistency).
- "This boundary is being deleted in the next PR — the diff intentionally breaks it." → Defer with an explicit link to the deletion PR.

These all share a property: they are *real engineering constraints*, not arguments that the test isn't worth writing. The deferral file records the constraint; the next reviewer can audit whether the constraint still holds.
