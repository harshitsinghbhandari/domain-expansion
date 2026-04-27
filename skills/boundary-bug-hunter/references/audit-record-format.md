# Audit Record Formats

The skill maintains three repo-committed audit artifacts. All three live at the repo root and use the same `seamHash` algorithm so they can be cross-referenced.

| File | Purpose | When written |
|------|---------|--------------|
| `.boundary-deferrals.json` | Explicit out-of-scope decisions | When the user defers a specific boundary with a written reason |
| `.boundary-overrides.log` | Emergency wave-throughs | When `BOUNDARY_OVERRIDE`/`--override` is used |
| `.boundary-verifications.json` | Verification cache | After every successful run; speeds up re-runs |

This file specifies all three. Deferrals get the most coverage because they're the most human-judgment-heavy.

---

## Deferrals — `.boundary-deferrals.json`

When the user explicitly marks a boundary out of scope, the skill records the decision so it remains auditable in the next review cycle. The record is committed to the repo. Deferrals not in version control are not deferrals.

---

## File location

`.boundary-deferrals.json` at the repo root. The file is committed; reviewers can see it in diff form.

If a project has multiple sub-projects with independent ownership, the file may live at sub-project roots instead. The skill discovers the closest ancestor `.boundary-deferrals.json` to each boundary and reads that one.

---

## Schema

```json
{
  "schemaVersion": 1,
  "deferrals": [
    {
      "id": "<uuid>",
      "createdAt": "2026-04-27T15:32:01Z",
      "createdBy": "<git config user.email or login>",
      "producer": {
        "file": "src/payments/charge.ts",
        "function": "createCharge",
        "lineRange": [42, 78]
      },
      "consumer": {
        "file": "src/billing/record.ts",
        "function": "recordCharge",
        "lineRange": [11, 27]
      },
      "boundaryType": "cross-module-call",
      "contract": "createCharge returns { id: string, amountCents: number, currency: string, status: 'pending' | 'succeeded' | 'failed' }; recordCharge requires id, amountCents, status",
      "reason": "Stripe is the upstream; we cannot run it in CI. Contract verified manually via Stripe sandbox in QA pipeline (link: https://internal.example/qa/stripe-contract).",
      "expiresAt": "2026-10-27T00:00:00Z",
      "seamHash": "sha256:9a4c2b...e1f3",
      "lastSeenInDiff": "abc1234"
    }
  ]
}
```

### Field meanings

| Field | Required | Notes |
|-------|----------|-------|
| `schemaVersion` | yes | Current is `1`. Bump on incompatible changes. |
| `id` | yes | UUID v4. Used to reference the deferral in reports. |
| `createdAt` | yes | ISO-8601 UTC. When the deferral was first added. |
| `createdBy` | yes | `git config user.email`, or login from the host (`gh api user --jq .login` if available). The skill records the author so reviewers know who to ask. |
| `producer` | yes | `file`, `function`, `lineRange` (inclusive). The function is the resolved name at deferral time. |
| `consumer` | yes | Same shape as `producer`. |
| `boundaryType` | yes | One of: `cross-module-call`, `file-io`, `serialization`, `network-ipc`, `shared-state`, `schema-version`. |
| `contract` | yes | Human-readable description of what producer claims and consumer expects. The skill uses this in the seam hash. |
| `reason` | yes | Free-text written reason. Must be substantive — not "later" or "skip". The skill rejects empty or sub-30-character reasons. |
| `expiresAt` | optional | ISO-8601 UTC. After this date the deferral becomes stale even if the seam hasn't changed. Default: 6 months from `createdAt` if omitted. |
| `seamHash` | yes | Computed at deferral time. See "Seam hash algorithm" below. |
| `lastSeenInDiff` | yes | Short SHA of the most recent diff in which the seam was observed. Updated by the skill on each run that sees the seam. |

---

## Seam hash algorithm

The seam hash binds the deferral to the *structural identity of the seam*, not to any single commit. A deferral remains valid as long as the producer and consumer functions still exist with the same identity and the contract description is unchanged.

```
seamHash = sha256(
  normalize(producer.file) + ":" + producer.function + "::" +
  normalize(consumer.file) + ":" + consumer.function + "::" +
  normalize(contract)
)
```

Where:

- `normalize(file)` resolves to a repo-relative POSIX path (forward slashes, no leading `./`).
- `normalize(contract)` collapses whitespace, lowercases, and trims surrounding punctuation. The contract is a normalized string so that minor wording edits don't invalidate the deferral but real contract changes do.

**Why hash this and not the commit:**
- Hashing the commit invalidates every deferral on every push. Useless.
- Hashing the file content invalidates on every reformat. Too brittle.
- Hashing structural identity (file + function + contract) tracks the *seam*, which is the thing the deferral is actually about. The seam survives renames within the function body and reformats; it does not survive renaming the function or moving it to a different module — both of which are changes that reasonable reviewers would want to re-examine.

**File renames:** When the producer or consumer file is renamed and the skill detects it via `git log --follow`, it offers to update the deferral's `file` field and recompute the hash. The user must explicitly accept; silent updates would defeat the audit purpose.

---

## Lifecycle

### Add a deferral

When the user marks a boundary out of scope:

1. The skill prompts for a substantive `reason` (rejects under 30 characters or generic responses).
2. The skill computes the `seamHash` from current producer/consumer/contract.
3. The skill appends a record to `.boundary-deferrals.json` with `id` (new UUID), `createdAt` (now), `createdBy` (from git config), and the user's reason.
4. The skill stages the file change and informs the user: "Deferral recorded. Commit `.boundary-deferrals.json` so reviewers see it."
5. The skill does NOT commit autonomously — the user owns the commit.

### Apply a deferral on subsequent runs

For every boundary the skill discovers in a new run:

1. Compute the boundary's current seam hash.
2. Look up the hash in `.boundary-deferrals.json`.
3. **Match found:** Skip the boundary. In the report's "Deferrals applied this run" section, list it with `createdAt`, `createdBy`, `reason`. Update `lastSeenInDiff` to the current diff's head SHA.
4. **No match found, but the same `(producer.file, producer.function, consumer.file, consumer.function)` tuple matches a deferral with a *different* hash:** The seam structure is the same but the contract description has drifted. Mark the deferral **invalidated**. Surface in "Stale deferrals" section. Require fresh decision.
5. **No structural match at all:** The boundary is new. Verify normally.

### Detect stale deferrals

For every record in `.boundary-deferrals.json`:

- If `expiresAt` is in the past → stale. Surface in report.
- If the recorded `(producer.file, producer.function)` no longer exists in the codebase → stale (producer was deleted or renamed). Surface and suggest the user remove the record.
- If the recorded `(consumer.file, consumer.function)` no longer exists → stale. Same.
- If `lastSeenInDiff` is more than 90 days behind the latest commit on the default branch → stale (the seam may have been refactored away; recheck). Surface.

Stale deferrals do not cause `boundary-status.json.status` to flip to `fail` on their own, but they appear in the report's "Stale deferrals (please clean up)" section and the user is expected to remove them in the same PR.

### Delete a deferral

The user removes the record from `.boundary-deferrals.json` manually. The skill does not delete records autonomously.

If a removed deferral's seam still exists and is unverified, the next run treats it as a normal unverified boundary and fails the gate.

---

## Examples

### Legitimate deferral

```json
{
  "id": "f1c8a2d4-7b9e-4a3f-8c12-9e5b3d7a1c4f",
  "createdAt": "2026-04-27T15:32:01Z",
  "createdBy": "harshit@example.com",
  "producer": {
    "file": "src/payments/charge.ts",
    "function": "createCharge",
    "lineRange": [42, 78]
  },
  "consumer": {
    "file": "external:stripe-api",
    "function": "POST /v1/charges",
    "lineRange": [0, 0]
  },
  "boundaryType": "network-ipc",
  "contract": "POST /v1/charges with idempotency-key header; response is { id, status, amount }; consumer expects 200 + json body or documented Stripe error envelope",
  "reason": "Stripe is the upstream; we cannot run it in CI. Contract verified via Stripe's published OpenAPI and our nightly Stripe-sandbox integration suite (job: nightly-stripe-contract). Re-evaluate when Stripe rotates API version.",
  "expiresAt": "2026-10-27T00:00:00Z",
  "seamHash": "sha256:9a4c2b8f3e1d7a6c2b5f8e9d4a3c7b1e6f2d5a9c8b3e7f1a4d6c2b8e5f3a9d7c",
  "lastSeenInDiff": "abc1234"
}
```

### Rejected deferral (substantive reason missing)

```json
{
  "reason": "later"
}
```

The skill rejects this with: `Deferral reason too short or non-substantive. Provide a written reason that captures the engineering constraint requiring the deferral. Examples: which external system blocks CI access, which infrastructure is missing, which subsequent PR will remove the seam.`

---

## Auditing the file

`.boundary-deferrals.json` is a first-class review artifact. When `pr-review` runs and sees changes to this file, it should:

- Flag every added deferral for explicit reviewer scrutiny.
- Quote the `reason` in the review summary.
- Confirm the `expiresAt` is reasonable (default 6 months; longer requires justification).
- Verify the `createdBy` matches the PR author or that the PR description explains why someone else's deferral is being added.

A growing `.boundary-deferrals.json` over time is a leading indicator of accumulating risk. Quarterly review: scan for deferrals whose `expiresAt` is approaching and whose underlying constraints may have changed.

---

## Overrides — `.boundary-overrides.log`

When the user invokes the skill with `BOUNDARY_OVERRIDE="<reason>"` or `--override "<reason>"`, the skill waves through unverified boundaries, sets `boundary-status.json.status` to `pass`, and appends an entry to this log. The log is committed to the repo so override use is visible in PR diffs.

### Format

Append-only JSON Lines. One JSON object per line. Never edit prior lines.

```jsonl
{"id":"2026-04-27T15:32:01Z-abc1234","timestamp":"2026-04-27T15:32:01Z","commitSha":"abc1234","author":"harshit@example.com","reason":"Payments outage; Stripe is rejecting webhooks. Hotfixing the signature verification; round-trip test added in follow-up PR #248. Will revisit by 2026-05-04.","branch":"hotfix/stripe-sig","wavedThroughBoundaries":[{"seamHash":"sha256:9a4c2b...e1f3","producer":"src/payments/webhook.ts:verifySig:42-78","consumer":"src/payments/webhook.ts:handleEvent:80-120","boundaryType":"cross-module-call"}]}
```

### Field meanings

| Field | Required | Notes |
|-------|----------|-------|
| `id` | yes | `<ISO-8601>-<short-sha>`. Globally unique. |
| `timestamp` | yes | When the override was used. |
| `commitSha` | yes | The HEAD SHA at the moment of override. |
| `author` | yes | From git config. |
| `reason` | yes | Mandatory. Rejected if under 30 characters or generic ("hotfix", "later", "skip"). Must explain the engineering constraint and the planned follow-up. |
| `branch` | yes | The branch name. |
| `wavedThroughBoundaries` | yes | Every boundary that was waved through, with its seam hash so the audit-overrides mode can cross-reference current state. |

### Rate limit (`--max-overrides-per-month`)

Soft enforcement. The skill counts entries from the last 30 days for the same `branch` (or for the repo if the branch is `main`/default). When the count exceeds the limit:

- The override is rejected for this run.
- `boundary-status.json.status` becomes `fail`.
- A loud message: `OVERRIDE LIMIT EXCEEDED: <N> overrides used in last 30 days (limit <M>). Address the boundaries or raise the limit explicitly in .boundary-bug-hunter.json.`

Default limit is 5/month for repos with no `.boundary-bug-hunter.json` config. Teams that find this restrictive should increase the limit explicitly — the goal is visibility, not paternalism.

### Audit-overrides mode

`boundary-bug-hunter audit-overrides` reads every entry since the last audit and produces `boundary-overrides-audit.md`:

```markdown
# Override Audit — <YYYY-MM-DD>

Audited <N> entries from <last-audit-date> to now.

## Followed up (boundary now verified or deferred)

| Override id | Boundary | Verified by | Verified on |
|-------------|----------|-------------|-------------|
| ...         | ...      | round-trip test in `tests/integration/x.ts` | <PR / commit> |

## Outstanding (still unverified, still no deferral)

| Override id | Boundary | Used reason | Days outstanding |
|-------------|----------|-------------|------------------|
| ...         | ...      | ...         | <N>              |

## Override usage trend

<N>/month over the last <K> months. Trend: increasing | flat | decreasing.
```

The audit timestamp is recorded in `.boundary-bug-hunter.json` so the next audit picks up where this one left off. Outstanding entries that are >30 days old are flagged red — they represent debt that was never paid.

---

## Verification cache — `.boundary-verifications.json`

Speeds up re-runs by caching boundary-verification status keyed on `(seamHash, testFileHash)`. Without this cache, every run re-derives coverage by reading every test file. With it, only changed seams or changed tests are re-checked.

### Schema

```json
{
  "schemaVersion": 1,
  "verifications": [
    {
      "seamHash": "sha256:9a4c2b...e1f3",
      "method": "round-trip" | "contract" | "replay" | "property",
      "testFile": "tests/integration/charge_test.ts",
      "testFileHash": "sha256:1f3e9d...4a2c",
      "verifiedAt": "2026-04-27T15:32:01Z"
    }
  ]
}
```

### Lifecycle

- **Cache hit** when both `seamHash` and `testFileHash` match → boundary is verified, skip re-derivation.
- **Cache miss** when either hash changed → re-derive coverage from scratch and update the cache entry.
- **Stale entry** when the cached `testFile` no longer exists or the cached `seamHash` no longer corresponds to any boundary on the current diff → drop the entry on next write.

The cache is committed (so CI doesn't re-derive cold every run). The skill never trusts the cache for the final report — it always validates that the cached test file still exists and that the cached method's invariant still holds. The cache is a hint, not an authority.

A cold cache (no file present, or schemaVersion mismatch) is a normal first run — it is not an error.
