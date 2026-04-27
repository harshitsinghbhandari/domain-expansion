# Accepted Verification Methods

A boundary is **verified** when at least one of these methods exercises the seam end-to-end. Round-trip is the gold standard, but it is not the only acceptable form. Many real seams have one side that genuinely cannot run in-process — third-party APIs, blob storage, hardware, billing providers, message brokers on dedicated infra. For those, contract tests, recorded-replay, and seam property tests provide real coverage without the deferral becoming a paperwork exercise.

This file specifies the four methods, when to choose which, and what each generated stub looks like.

A boundary is **unverified** when none of the four exists. Separate unit tests, mocked downstream calls, manual QA, "it works in staging", and a documented contract that no test executes all count as unverified.

---

## 1. Round-trip test (gold standard)

**What it is:** A single test runs the real producer and the real consumer back-to-back. The test asserts the contract survives end-to-end.

**Choose when:** Both sides of the seam can run in-process or with the project's existing integration-test fixtures (testcontainers, in-memory adapters, real database with rollback, real internal queue).

**Strengths:**
- Catches every shape of contract drift: schema mismatch, type drift, encoding loss, stripped fields, default-value confusion.
- Future-proof: the next change to either side runs against the real other side, so drift fails fast.

**Limits:**
- Cannot verify seams against external systems (Stripe, S3, payment processors, hardware).

**Stub patterns:** See [test-frameworks.md](test-frameworks.md) for per-language templates.

---

## 2. Consumer-driven contract test

**What it is:** A contract artifact (a fixture file) pins the shape the consumer expects. Two tests share the artifact:
- **Producer-side test:** asserts the producer's real output conforms to the contract artifact.
- **Consumer-side test:** asserts the consumer correctly handles the contract artifact as input.

The two halves never run in the same process. Drift in either side fails one of the two tests.

**Choose when:** The producer or consumer cannot run in-process. Common cases: external HTTP API your code calls, internal microservice with a separate test environment, message broker your code publishes to.

**Strengths:**
- Catches structural drift on either side without requiring both to run together.
- The contract artifact is committable, reviewable, and forces an explicit version bump on intentional changes.
- Aligns with established patterns (Pact, OpenAPI contract tests, Avro schema registry).

**Limits:**
- Catches structural drift, not behavioral drift. If the producer changes from "always returns USD" to "returns the user's currency" without changing the structural shape, neither test fails.
- Requires discipline: the contract artifact must be updated in the same PR as the change that causes drift, or the tests pass while production breaks.

### Stub pattern (TypeScript / Vitest example)

Contract artifact (`fixtures/charge-response.json`):

```json
{ "id": "ch_123", "amountCents": 1000, "currency": "USD", "status": "succeeded" }
```

Producer-side test (`tests/contract/charge-producer.test.ts`):

```ts
import { describe, it, expect } from 'vitest';
import { createCharge } from '../../src/payments/charge';
import contract from '../../fixtures/charge-response.json';

describe('contract: createCharge → charge-response', () => {
  it('produces output conforming to the contract', async () => {
    const produced = await createCharge({ amountCents: 1000, currency: 'USD' });
    expect(Object.keys(produced).sort()).toEqual(Object.keys(contract).sort());
    expect(typeof produced.id).toBe(typeof contract.id);
    expect(typeof produced.amountCents).toBe(typeof contract.amountCents);
    expect(['pending', 'succeeded', 'failed']).toContain(produced.status);
  });
});
```

Consumer-side test (`tests/contract/charge-consumer.test.ts`):

```ts
import { describe, it, expect } from 'vitest';
import { recordCharge } from '../../src/billing/record';
import contract from '../../fixtures/charge-response.json';

describe('contract: charge-response → recordCharge', () => {
  it('consumer handles the contract shape end-to-end', async () => {
    const result = await recordCharge(contract);
    expect(result.recorded).toBe(true);
    expect(result.chargeId).toBe(contract.id);
  });
});
```

Generated stub for any language follows the same shape: one fixture, two tests sharing it.

---

## 3. Recorded-replay test

**What it is:** The producer's real output is captured once (live call, sandbox call) and stored as a fixture. Tests replay the fixture into the real consumer. The next change to the producer that re-records reveals drift via the fixture diff.

**Choose when:** The producer is a slow, expensive, rate-limited, or external system but its output shape is stable enough to commit. Common cases: third-party API responses, ML model output, expensive computation, browser-rendered output.

**Strengths:**
- Captures the producer's real output, including edge cases the test author wouldn't think to write.
- Cheap to maintain — re-record on intentional change, otherwise the fixture is static.
- Fixture diffs become reviewable: every PR that re-records shows reviewers exactly what changed in the producer's output.

**Limits:**
- Captures one snapshot of producer behavior. Variations (different inputs, time-dependent fields, randomness) need separate fixtures.
- The fixture can drift from production reality if not periodically re-recorded.
- Sensitive fields in the recording (tokens, PII) need redaction.

### Stub pattern

Capture script (`scripts/record-charge-fixture.ts`):

```ts
// Run manually or in a scheduled job to refresh the fixture.
// Redacts PII; commits the result to fixtures/.
import { writeFileSync } from 'node:fs';
import { createChargeAgainstSandbox } from '../src/payments/charge';
import { redact } from '../tests/helpers/redact';

const recorded = await createChargeAgainstSandbox({ amountCents: 1000, currency: 'USD' });
writeFileSync('fixtures/recorded/charge-success.json', JSON.stringify(redact(recorded), null, 2));
```

Replay test (`tests/replay/charge-replay.test.ts`):

```ts
import { describe, it, expect } from 'vitest';
import { recordCharge } from '../../src/billing/record';
import recorded from '../../fixtures/recorded/charge-success.json';

describe('replay: recorded charge → recordCharge', () => {
  it('consumer handles the recorded producer output', async () => {
    const result = await recordCharge(recorded);
    expect(result.recorded).toBe(true);
  });
});
```

A README in `fixtures/recorded/` should document: when each fixture was last refreshed, the command to re-record, and any redactions applied.

---

## 4. Seam property test

**What it is:** A generator produces inputs satisfying the producer's contract; the test asserts the consumer never crashes / never returns invalid shapes / always satisfies a stated invariant.

**Choose when:** The contract is wide (many shapes, many enum values, many optional fields) and the assertion is "must not surprise the consumer", not "specific output for specific input". Common cases: parser/serializer pairs, schema validators, stateful state machines with many transitions.

**Strengths:**
- Explores the long tail of contract values that hand-written tests would miss.
- Surfaces consumer assumptions that aren't documented anywhere (e.g., "consumer assumes `status` is non-empty even though the contract says optional").

**Limits:**
- Catches "consumer breaks" but not "consumer silently produces wrong output". Hand-written assertions are still needed for value-level correctness.
- Generator quality matters — a generator that always produces the same shape catches nothing.

### Stub pattern (TypeScript / fast-check)

```ts
import { describe, it } from 'vitest';
import fc from 'fast-check';
import { recordCharge } from '../../src/billing/record';

describe('seam-property: any contract-conforming charge → recordCharge', () => {
  it('consumer never crashes on contract-conforming input', () => {
    fc.assert(
      fc.property(
        fc.record({
          id: fc.string({ minLength: 1, maxLength: 64 }),
          amountCents: fc.integer({ min: 1, max: 1_000_000_000 }),
          currency: fc.constantFrom('USD', 'EUR', 'GBP', 'JPY'),
          status: fc.constantFrom('pending', 'succeeded', 'failed'),
        }),
        async (input) => {
          const result = await recordCharge(input);
          // Invariant: every accepted input produces a result with a recorded chargeId.
          if (result.recorded && !result.chargeId) {
            throw new Error('invariant violated: recorded but no chargeId');
          }
        },
      ),
      { numRuns: 200 },
    );
  });
});
```

Use the project's existing property-test framework if one is in place: `fast-check` (TS/JS), `hypothesis` (Python), `proptest`/`quickcheck` (Rust), `gopter` (Go), `Hedgehog` (Haskell, F#), `jqwik` (Java/Kotlin), `FsCheck` (.NET).

---

## Choosing between methods (decision rules)

```
Both sides run in-process?
  Yes → round-trip test.
  No → does the producer's output shape stabilize meaningfully?
    Yes → consumer-driven contract test (preferred when the team owns both sides).
    No → recorded-replay test (preferred for genuinely external producers).

Is the contract wide and the assertion mostly "must not crash / must not produce invalid shape"?
  Yes → add a seam property test in addition to one of the above.
```

A boundary may have multiple verification artifacts. Prefer one strong artifact over many weak ones, but a round-trip + contract test combination is common and welcome at high-risk seams (payments, auth, persistence).

## What does NOT count as verification

For clarity:

- **Mocked consumer / mocked producer**, even with realistic mock responses. Mocks are agreements between the test author and themselves — not between producer and consumer.
- **Type-system-only proofs.** TypeScript types passing does not prove the runtime values match, especially across serialization boundaries.
- **Schema definitions with no executing test.** A documented JSON schema, OpenAPI spec, or Zod object that nothing runs against verifies nothing.
- **"It works in staging"** — staging finds symptoms downstream of the bug. The verification artifact catches it before it ships.
- **Manual QA pass** — non-reproducible, doesn't run on the next change.
