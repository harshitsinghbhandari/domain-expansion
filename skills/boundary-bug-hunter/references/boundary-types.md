# Boundary Types — Field Guide

The eight recognized boundary types, with patterns to look for, the contract that must be checked, and the shape of a passing round-trip test.

When in doubt about whether something is a boundary: if a piece of code passes data, control, or state to another piece of code that was tested separately, it's a boundary.

---

## 1. Cross-module function call

**Definition:** A function in module A calls a function in module B (or another file, package, namespace, crate). The call crosses a unit-test boundary because A and B are typically tested independently.

**Patterns:**
- Imports across packages / files / modules.
- Re-exports through a barrel file or facade.
- Plugin / hook callbacks where the caller and callee live in different repos or packages.

**Contract to check:**
- **Argument shape**: What does A pass? What does B expect? Are types nominally identical or just structurally similar (TypeScript structural vs nominal types, Python duck-typing)?
- **Return shape**: What does B return? Does A handle every return variant — `null`, `undefined`, error sentinels, empty collections, `Maybe`/`Option`/`Result`?
- **Throws / errors**: What can B raise? Does A catch the right types? In statically typed languages, are checked exceptions or `Result` variants exhaustively handled?
- **Side effects**: Does B mutate the input? Modify globals? A's correctness may depend on B not doing either.

**Round-trip test:**
Call B through A with realistic input, assert A's output reflects what B actually returned. Do not mock B.

**Common bugs hiding here:**
- A's tests mock B; B's tests test B in isolation; the real combined call has never run.
- B was changed to return `T | undefined`, A still assumes `T`.
- B's argument shape evolved (added required field), A still calls with the old shape; TypeScript/etc. catch some but not all.

---

## 2. File I/O

**Definition:** Code writes to a file, another piece of code (sometimes the same code on a later run) reads it. The file is the seam.

**Patterns:**
- Configuration files written by one tool, read by another.
- Cache files, lock files, sentinel files (`.lock`, `.pid`, `.last-run`).
- Database files (SQLite, embedded KV stores).
- Atomic-write patterns: `write to .tmp → fsync → rename to final`.
- State files used to coordinate between processes (workers, daemons).

**Contract to check:**
- **Format**: Same JSON/YAML/binary schema on both sides? Encoder library matches decoder library?
- **Encoding**: UTF-8 vs UTF-16, BOM, line endings, trailing newline.
- **Atomicity**: If the writer crashes mid-write, can the reader handle the partial file? Is there a `.tmp + rename` pattern?
- **Cleanup**: Does anyone delete temp files on failure? Lock files on crash?
- **Concurrent access**: Two writers? Reader during write?

**Round-trip test:**
Producer writes to a tempdir; consumer reads from the same path; assert the read value equals the original. Add a test for partial-write recovery if the writer is non-atomic.

**Common bugs hiding here:**
- Writer uses `JSON.stringify` (no replacer for `Date`); reader uses `JSON.parse` (gets strings, not Dates) — silent type drift.
- New field added to writer; reader's parse schema rejects unknown keys (Pydantic strict mode, Zod `.strict()`).
- Atomic-write rename works; cleanup of `.tmp` on rename failure forgotten; tmpfiles accumulate.

---

## 3. Serialization / parsing

**Definition:** A value is serialized to bytes (JSON, YAML, Protobuf, MessagePack, Avro, custom binary, URL-encoded form, etc.) and later parsed back. The encoder and decoder are the two sides of the seam, even when they're in the same module.

**Patterns:**
- API request/response bodies.
- Event/message payloads on a queue.
- Config marshalling.
- Caching serialized objects.
- Deep-clone-via-serialize patterns (`JSON.parse(JSON.stringify(x))`).

**Contract to check:**
- **Schema parity**: Compile-time type vs runtime validator. If both exist (TypeScript + Zod, Python + Pydantic), every field in the type must exist in the validator. Fields in the type missing from the validator are silently stripped on parse.
- **Type fidelity**: `Date` survives? `BigInt` survives? `undefined` vs `null` round-trips correctly? `NaN`/`Infinity` handled?
- **Optional vs nullable**: Producer omits a field — does the consumer see `undefined`, `null`, or a default?
- **Forward/backward compatibility**: New optional field added by producer — does an old decoder ignore it (good) or reject it (bad)? Old field removed — does a new decoder default safely?
- **Enum exhaustiveness**: Producer writes a value the consumer doesn't recognize — fails closed (rejects) or fails open (accepts and silently mishandles)?

**Round-trip test:**
`encode(decode(encode(value)))` equals `encode(value)` for representative inputs. For schema parity, an explicit assertion that every field in the static type is also in the runtime schema (a meta-test).

**Common bugs hiding here:**
- `Date` → JSON → string → `new Date(string)` works in test data but breaks on timezones with no `Z` suffix.
- Zod `.object({...})` with no `.strict()` silently accepts extra fields; producer adds field with same name as something else, consumer overwrites silently.
- Protobuf `optional` vs unset — old proto2/proto3 distinction trips up consumers expecting field presence.

---

## 4. Network / IPC / signals

**Definition:** Code on one side of a process/network boundary sends a message; code on the other side receives it. The wire format is the contract.

**Patterns:**
- HTTP / gRPC clients and servers.
- WebSockets, server-sent events.
- Message queues (Kafka, RabbitMQ, SQS, BullMQ, Sidekiq, Celery).
- Inter-process communication: pipes, sockets, shared memory, signals (`SIGTERM` and friends).
- WebHooks (the producer is external — even higher contract drift risk).

**Contract to check:**
- **Wire format**: JSON / Protobuf / Avro / form-encoded / binary. Same library, same version, same options on both sides.
- **HTTP semantics**: Status codes — does the consumer treat 5xx the same as 4xx? Retry logic — idempotent endpoints only? Headers (`Content-Type`, custom headers) preserved across proxies?
- **Queue delivery semantics**: At-least-once means consumer must be idempotent; at-most-once means producer must handle dropped messages.
- **Backpressure / rate limits**: Producer doesn't overwhelm consumer? Consumer signals when overloaded?
- **Auth context**: Token / cookie propagated correctly? Refresh handled?
- **Signal handling**: `SIGTERM` triggers graceful shutdown? In-flight work drained before exit?

**Round-trip test:**
Use the project's existing integration-test infrastructure (testcontainers, in-process server, mock queue). Send a message through the real producer, receive it through the real consumer, assert the payload survives and the consumer's side effect occurs. For idempotency: call the consumer twice with the same message, assert the side effect occurs once.

**Common bugs hiding here:**
- Client uses `application/json`, server expects `application/x-www-form-urlencoded` (or vice versa). Each works in isolation; combined fails.
- Consumer is not idempotent; queue redelivers on consumer crash; duplicate side effects.
- Timeout on producer is shorter than processing time on consumer; producer retries; consumer processes both — duplicate work.
- Webhook signature validated with the wrong shared secret after rotation; producer signs with new, consumer validates with old.

---

## 5. Shared mutable state

**Definition:** A piece of state that lives outside any single function and is read or written by multiple code paths. The "boundary" is implicit — the state itself is the seam.

**Patterns:**
- Global variables, singletons, module-level state.
- Caches (in-memory, Redis, file-backed).
- Registries (plugin lists, observer subscriptions).
- Lock files, PID files.
- Environment variables read at runtime (the OS is the producer).
- Database rows used as inter-service flags.

**Contract to check:**
- **Write visibility**: Writer mutates state; reader sees the new value — but only if reads happen after writes. What if the order is reversed (race)? In a multi-process system, what if cache invalidation lags?
- **Initialization order**: State is read before it's initialized? In Python, module-level state initialized at import time. In JS, top-level `await` and dynamic imports complicate this.
- **Cleanup**: State is set; never cleared. Subsequent reads see stale data. Test runs leak state between tests.
- **Concurrency**: Two writers race; resulting state is one writer's, not a merge. Reader sees inconsistent intermediate state during multi-field write.
- **Ownership**: Who is allowed to write? If "anyone", the state is a footgun. Document and enforce.

**Round-trip test:**
Sequence is the test. Write the state in one code path, read it in another, assert. For concurrency: write twice in parallel and assert one writer wins cleanly. For staleness: write, mutate underlying source, re-read, assert appropriate behavior (refresh? error?).

**Common bugs hiding here:**
- Test 1 sets a global; test 2 reads it; passes when run together, fails in isolation.
- Cache populated on first request; underlying data changes; cache TTL is 1 hour; reader sees stale answer for 59 minutes.
- Two writers both set `state.value = ...` from concurrent requests; only one survives; the loser's caller thinks their write succeeded.
- Process A writes a lock file and crashes; process B sees the lock and waits forever.

---

## 6. Schema versioning

**Definition:** The producer and consumer agree on a schema, but the schema evolves over time. At any moment, an old producer might write data a new consumer reads, or vice versa. The version skew is the seam.

**Patterns:**
- Database migrations (forward and rollback).
- Persisted snapshots / event-sourced events with version tags.
- API versioning (v1, v2 endpoints; or evolving same-endpoint payloads).
- Cached values that survive process restarts across deploys.
- Long-lived cookies / tokens encoding shape that has since changed.

**Contract to check:**
- **Forward compatibility**: New consumer reads old data. Does it default missing fields safely? Does it reject confidently or accept and corrupt?
- **Backward compatibility**: Old consumer reads new data. Does it ignore unknown fields? Or reject?
- **Migration symmetry**: Forward migration `up()` and rollback `down()` are inverses? Does `down()` correctly handle data created *after* `up()` ran (new rows in new format)?
- **Versioning signal**: Does the data carry a version field? Or does the consumer infer? Inferred versions are fragile.
- **Default handling**: New required field added; old data has no value; default chosen — is the default semantically correct, or just "non-null"?

**Round-trip test:**
- Forward: write with old encoder, read with new decoder, assert.
- Backward: write with new encoder, read with old decoder (if old decoder still in production), assert acceptance.
- Migration: apply `up()`, then `down()`, assert original state. Then apply `up()`, write new-format data, apply `down()`, assert *new* data is handled correctly (not silently dropped).

**Common bugs hiding here:**
- Migration `up()` adds a column with `DEFAULT NULL`; old code paths still write rows with the old shape; `down()` drops the column — fine. New code paths write the new column; `down()` drops the column — silent data loss the rollback didn't account for.
- Event-sourced events stored with version 1 schema; new code reads them assuming version 2 fields exist; silently uses default values that are wrong.
- API v2 returns a field as a number; clients on old SDKs expect string; coerce or crash depending on language.

---

---

## 7. Cross-language seam

**Definition:** Code in one language hands data to code in another. The static type system on each side stops at the language boundary. Drift is silent because no compiler sees both sides.

**Patterns:**
- Web frontend (TypeScript) calling backend (Python / Go / Ruby / etc.) over HTTP, gRPC, or GraphQL.
- Mobile client (Swift / Kotlin) calling REST API.
- Native extension (Rust / C / C++) called from a high-level runtime (Node / Python / Ruby).
- Polyglot monorepo with a shared protocol (Protobuf, OpenAPI, JSON Schema) supposedly enforced across services — but only enforced if both sides regenerate.
- WebAssembly module called from JavaScript host.
- Inter-process serialization (FFI, foreign function calls, language-bridged queues).

**Contract to check:**
- **Schema source of truth:** Is there one canonical schema (Protobuf `.proto`, OpenAPI YAML, JSON Schema, Avro) that both sides generate from? If not, the contract is "what each side happens to implement", which always drifts.
- **Generation freshness:** When the schema changes, do CI checks ensure both sides are regenerated and committed? Or can one side merge a schema change without the other catching up?
- **Type-system fidelity loss:** The schema may express things the target language can't (e.g., Protobuf `optional` semantics in TypeScript, Rust enums in Python). Where fidelity is lost, the consumer must explicitly handle the loss.
- **Runtime validation on parse:** Even with codegen, the receiver should validate at the language boundary. Trusting the wire format crashes when an old client talks to a new server (or vice versa).
- **Null / undefined / missing distinctions:** TypeScript distinguishes `null` from `undefined` from missing. Most other languages do not. JSON does not. Verify that the receiver's interpretation matches the sender's.

**Round-trip test:**
- **Within a polyglot monorepo:** integration test that boots both services and asserts a real round-trip across the wire. testcontainers + the project's service-runner is the standard pattern.
- **Across repos / SDKs:** consumer-driven contract test (see [verification-methods.md](verification-methods.md)) is usually the right tool. The schema artifact is the contract.
- **Native extension:** test that calls the native function with realistic inputs, asserts the high-level wrapper interprets the result correctly. Run on every supported runtime version.

**Common bugs hiding here:**
- TypeScript frontend serializes `Date` as ISO string; Python backend's Pydantic schema expects `datetime`; on some Python versions this works (auto-coerce), on others it 422s.
- Protobuf `optional` field added on the producer; consumer regenerated months ago and treats absence as default; default is wrong for the new field.
- Rust extension returns `Result<T, E>`; Python wrapper raises a generic exception; caller catches `Exception` and silently drops the specific failure mode.
- Mobile app uses an old version of the API; backend silently changed an enum value; mobile app crashes on parse OR worse, treats unknown enum as the default and proceeds.

---

## 8. Event-sourced / CQRS seam

**Definition:** Producer and consumer are separated in time as well as in space. The producer (command handler, event writer) emits events; the consumer (projection, read model, query handler, downstream subscriber) reads them — possibly hours, days, or releases later. Replay correctness, projection drift, and command/query divergence are all variants of this seam.

**Patterns:**
- Event-sourced systems (EventStore, Kafka with event-sourcing semantics, custom event log).
- CQRS architectures with command-side writes and query-side reads.
- Audit logs that downstream tools build dashboards on.
- Long-lived projections that get rebuilt from history.
- Outbox patterns where events are persisted in the same transaction as state changes.
- Saga / process-manager systems where a multi-step workflow's state is reconstructed from events.

**Contract to check:**
- **Event versioning:** Does each event carry an explicit `version` / `schemaVersion` field? Without it, a projection rebuild against historical events silently misinterprets old events as the current shape.
- **Backwards-compatible event evolution:** New event versions must be readable by old projection code (forward compat) or projection code must explicitly upcast old → new on read. Adding a required field to an event without a migration story breaks rebuild.
- **Replay determinism:** Replaying the event log from the beginning must produce the same projection state as live processing. Side effects in projection handlers (HTTP calls, sending emails) violate this and silently corrupt rebuilds.
- **Command/query divergence:** Command-side validation (write) and query-side filtering (read) often duplicate the same rules in different code. They must agree. If write rejects `status: cancelled` but read still surfaces cancelled rows, the system has lied to itself.
- **Idempotency on consumer:** At-least-once delivery is the norm. Each event must be processable multiple times without producing duplicate side effects. Look for: idempotency keys, dedupe tables, idempotent state mutations.
- **Out-of-order delivery:** Where ordering is not guaranteed (parallel partitions, retries), the consumer must tolerate or detect out-of-order events.

**Round-trip test:**
- **Replay correctness:** Snapshot a representative event sequence as a fixture; replay it through the projection; assert final state. Then re-run with a perturbation (skip an event, duplicate one, reorder two) and assert the projection handles each case correctly.
- **Versioned-event compatibility:** For every event version, store a fixture event of that version; assert current projection code can process it.
- **Command + query symmetry:** A property test that drives commands through the write side, then queries through the read side, asserts the resulting view matches the executed commands.
- **Idempotency:** Apply the same event twice in a row; assert state is identical to applying it once.

**Common bugs hiding here:**
- Event V1 had field `userId`; V2 renamed to `actorId` with a migration; projection code still reads `userId` and silently sees `undefined` for V2 events.
- Projection handler calls a third-party API (sends an email). Replay against history sends every email a second time.
- Command side validates "amount > 0"; query side filters by "amount >= 0". A bug introduces a `0` somewhere; write rejects, read surfaces it from old data; product becomes inconsistent.
- New event type added; old projection rebuilds skip it (no handler) — silently lossy until someone notices the metric is wrong.
- Out-of-order delivery in a partitioned queue: `UserCreated` arrives after `UserDeleted`; consumer creates a deleted user that nothing else sees.

---

## How to detect boundary type from a code change

When triaging a diff, classify each change by which boundary type it touches:

| Change | Likely boundary |
|--------|-----------------|
| Modified function signature in an exported module | Cross-module function call |
| New JSON field, new schema, modified parser | Serialization |
| Modified file format, new sentinel/lock file | File I/O |
| New endpoint, modified payload, new event | Network/IPC |
| Modified global, cache, or singleton | Shared mutable state |
| New migration, version bump on persisted shape | Schema versioning |
| Modified shared schema artifact (`.proto`, OpenAPI, JSON Schema), modified codegen output | Cross-language seam |
| New event type, modified event handler, modified projection, replay-relevant change | Event-sourced / CQRS seam |

A single change frequently touches multiple boundaries. A migration that adds a column also adds a serialization field and likely a network payload field. Check all relevant boundaries, not just the most obvious one.
