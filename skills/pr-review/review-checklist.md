# PR Review Checklist

This checklist covers areas that CI cannot check. Skip items related to linting, formatting, type checking, and import ordering — those are handled by automated tools.

## Code Quality

### Abstractions and Design

- [ ] **Clear abstractions** - State management is explicit; no dynamic attribute setting/getting
- [ ] **No side-channel communication** - If behavior changes based on a hidden flag or dynamically-set attribute, the interface itself should change instead (different function signature, different class, different code path). Side-channel patterns (set a private flag in one place, check it in another via reflection) create undocumented behavioral modes
- [ ] **Proper interface, not on/off flags** - A private boolean that switches between two fundamentally different behaviors should be two separate code paths or a proper interface change, not a flag
- [ ] **Interface documentation** - New internal calling conventions, protocols, or contracts between components must have concrete documentation: what the caller provides, what the callee receives, what invariants hold, and cleanup responsibilities. Motivational comments ("this allows X") are not interface documentation
- [ ] **Match existing patterns in the same file** - Before accepting new code in a file, read how similar features are already implemented in that same file. If the file uses class attributes for boolean flags, new boolean flags must use class attributes. If the file uses a specific setter pattern, new setters must use the same pattern
- [ ] **No over-engineering** - Only requested changes are made; no speculative features
- [ ] **No premature abstraction** - Helpers and utilities are only created when reused; three similar lines is better than a one-use helper
- [ ] **No trivial helpers** - Avoid 1-2 LOC helper functions used only once (unless significantly improves readability)

### API Design

When a PR introduces new API patterns, carefully evaluate the broader implications:

- [ ] **No flag-based internal access** - Reject patterns like `_internal=True` kwargs that gate internal functionality. These are confusing to reason about, impossible to document properly, and create BC headaches. Use a separate private function instead (e.g., `_my_internal_op()`)
- [ ] **Pattern already exists?** - Before accepting a new pattern, search the codebase to check if this pattern is already established. If not, the PR is introducing a new convention that needs stronger justification
- [ ] **Documentation implications** - Can this API be clearly documented? Flag-based access creates ambiguity about what is public vs private
- [ ] **BC implications going forward** - Will this pattern create future BC constraints?
- [ ] **Testing implications** - Does this pattern require awkward test patterns? Internal-only flags often lead to tests that use "forbidden" parameters
- [ ] **UX implications** - Is this pattern discoverable and understandable to users? Will it appear in autocomplete, type hints, or docs in confusing ways?
- [ ] **Event/payload shape is namespaced** — When composing event data from multiple sources, each source should have its own key (e.g., `data: { context, ...reactionFields }`) rather than being spread flat. Flat spread risks silent key collisions now or when either source adds a field later.
- [ ] **Schema version on enriched payloads** — If a payload adds new fields alongside deprecated ones, include a `schemaVersion` field so consumers can migrate.
- [ ] **Deprecated fields have a removal plan** — Duplicated fields kept for backward compatibility must have a tracking issue or version target for removal. A comment saying "intentional" is not a plan.

### Sort/Filter/Comparison

- [ ] **Sort/compare correctness** — For every `.sort()`, `.filter()`, or comparison in changed code: what happens at 0 items? 1 item? 10+ items? Does string comparison handle numeric components correctly (lexicographic `-10` < `-2` bug)? Does the comparator return a consistent total order?

### Code Clarity

- [ ] **Self-explanatory code** - Variable and function names convey intent; minimal comments needed
- [ ] **Useful comments only** - Comments explain non-obvious context that cannot be inferred locally. For large explanations, use well-named notes that can be referenced from multiple places
- [ ] **No backward-compatibility hacks** - Unused code is deleted completely, not renamed with underscores or marked with "removed" comments
- [ ] **Appropriate complexity** - Solutions are as simple as possible for the current requirements
- [ ] **Documentation shows correct patterns only** - Docs and markdown files should show the right way to do things directly, not anti-patterns followed by corrections. Code examples must have correct indentation, names, and syntax

### Initialization and Module Design

- [ ] **No fragile init ordering** - If multiple imports/calls must happen in a specific undocumented order, flag the design. Dependencies should be explicit or combined into a single entry point
- [ ] **Idempotent global state** - Registries and global lists that accumulate entries must handle multiple calls safely (no duplicate registration, clear cleanup story)

## Infrastructure & Framework Usage

When a PR touches code in the scope of any item below, **stop and investigate** whether the established infrastructure should be used.

### Use Framework Utilities

- [ ] **Use existing utilities** - PR implements functionality manually instead of using framework-provided utilities (e.g., manual string manipulation vs library functions, custom validation vs framework validation)
- [ ] **Use established patterns** - PR creates new patterns when the codebase has established ways of doing the same thing
- [ ] **Use proper error types** - PR uses generic exceptions when more specific ones exist in the framework or codebase
- [ ] **Use configuration systems** - PR hard-codes values that should come from configuration
- [ ] **Use logging frameworks** - PR uses print statements instead of the project's logging infrastructure
- [ ] **No coupling to library internals** - PR accesses private/underscore-prefixed attributes, internal modules, or undocumented APIs of third-party libraries. These break without notice on library upgrades

### Resource Management

- [ ] **RAII/Context managers** - PR manually manages resources (files, connections, locks) instead of using context managers or RAII patterns
- [ ] **Proper cleanup** - PR allocates resources without ensuring cleanup on all exit paths (success, exception, early return)
- [ ] **Connection pooling** - PR creates new connections repeatedly instead of using connection pools
- [ ] **Memory management** - PR holds references longer than needed or creates circular references

### Database & Storage

- [ ] **Prepared statements** - PR uses string formatting for SQL queries instead of parameterized queries (SQL injection risk)
- [ ] **Transaction handling** - PR performs multiple related database operations without proper transaction boundaries
- [ ] **Index awareness** - PR adds queries that would benefit from indexes without considering index implications
- [ ] **Batch operations** - PR makes N+1 queries instead of using batch/bulk operations

### HTTP & Networking

- [ ] **Timeout configuration** - PR makes network calls without timeout configuration
- [ ] **Retry logic** - PR handles transient failures without retry logic where appropriate
- [ ] **Connection reuse** - PR creates new connections for each request instead of reusing connections
- [ ] **Error handling** - PR doesn't handle common network errors (timeouts, connection refused, DNS failures)

### Async & Concurrency Patterns

- [ ] **Async consistency** - PR mixes sync and async code inappropriately
- [ ] **Proper awaiting** - PR creates async tasks without awaiting or tracking them
- [ ] **Concurrency primitives** - PR uses low-level primitives when higher-level abstractions exist (e.g., manual threading vs executor pools)

## Testing

### Test Existence

- [ ] **Tests exist** - New functionality has corresponding tests
- [ ] **Tests are in the right place** - Tests should be added to an existing test file next to other related tests
- [ ] **New test file is rare** - New test files should only be added for new major features

### Test Patterns

- [ ] **Use test framework idioms** - Tests use the project's established testing patterns and utilities
- [ ] **Proper test isolation** - Tests don't depend on execution order or shared mutable state
- [ ] **Parameterized tests** - PR duplicates test methods that differ only in input data instead of using parameterization
- [ ] **Fixtures and setup** - PR duplicates setup code across tests instead of using fixtures or setup methods
- [ ] **Descriptive test names** - Test method names describe what is being tested

### Test Quality

- [ ] **Edge cases covered** - Tests include boundary conditions, empty inputs, error cases
- [ ] **Error conditions tested** - Expected exceptions are tested with message verification, not just exception type
- [ ] **No duplicated test logic** - Similar tests share helper methods with different configurations
- [ ] **Deterministic tests** - Tests don't rely on timing, random values without seeds, or external state
- [ ] **Test data management** - Tests use appropriate test data (fixtures, factories, builders) not production data

### Test Coverage Adequacy

- [ ] **Happy path tested** - Normal successful execution paths are tested
- [ ] **Error paths tested** - Error conditions and exception handling are tested
- [ ] **Boundary conditions** - Edge cases like empty collections, null values, max values are tested
- [ ] **Integration points** - Interactions with external systems are tested (mocked or integration tests)

## Security

### Input Validation

- [ ] **All inputs validated** - User inputs, API parameters, file contents are validated before use
- [ ] **Injection prevention** - No string concatenation for SQL, shell commands, or similar; use parameterized queries and safe APIs
- [ ] **Path traversal prevention** - File paths from user input are validated to prevent directory traversal
- [ ] **Size limits** - Inputs have appropriate size limits to prevent resource exhaustion

### Authentication & Authorization

- [ ] **Auth checks present** - Protected operations verify authentication and authorization
- [ ] **No credential exposure** - Secrets are not logged, exposed in error messages, or committed to code
- [ ] **Secure defaults** - Default configurations are secure (e.g., HTTPS, restricted permissions)

### Data Protection

- [ ] **Sensitive data handling** - PII and sensitive data are handled appropriately (encryption, masking, minimal retention)
- [ ] **Secure communication** - Network communication uses encryption where appropriate
- [ ] **No secrets in code** - API keys, passwords, tokens are not hardcoded

### Dependency Security

- [ ] **Dependency review** - New dependencies are from trusted sources and actively maintained
- [ ] **Version pinning** - Dependencies use specific versions, not ranges that could pull malicious updates
- [ ] **No unnecessary dependencies** - New dependencies are justified and not duplicating existing functionality

## Thread Safety & Concurrency

### Shared State

- [ ] **No unprotected shared mutable state** - Shared data structures accessed from multiple threads are protected by locks or are inherently thread-safe
- [ ] **Lock ordering** - When multiple locks are acquired, ordering is consistent to avoid deadlocks
- [ ] **No race conditions** - Operations on shared state that need to be atomic are properly synchronized

### Concurrency Patterns

- [ ] **RAII lock guards** - Prefer automatic lock management over manual lock/unlock to ensure exception-safe unlocking
- [ ] **Correct atomic usage** - Atomic operations use appropriate memory ordering
- [ ] **Thread-safe collections** - Use thread-safe collections when shared across threads

### Async Safety

- [ ] **No blocking in async context** - Blocking operations are not performed in async contexts without proper handling
- [ ] **Proper cancellation handling** - Long-running operations support cancellation
- [ ] **Resource cleanup on cancellation** - Resources are cleaned up when operations are cancelled

## Performance

### Obvious Regressions

- [ ] **No unnecessary allocations** - Objects are not repeatedly created in hot loops
- [ ] **Appropriate data structures** - Data structures match access patterns (e.g., maps for lookups, arrays for iteration)
- [ ] **No expensive operations in loops** - I/O, network calls, or expensive computations are not inside tight loops
- [ ] **Lazy initialization** - Expensive objects are initialized only when needed

### Algorithm Complexity

- [ ] **Appropriate complexity** - Algorithm complexity matches the problem (no O(n²) when O(n) is possible)
- [ ] **Pagination** - Large result sets are paginated rather than loaded entirely into memory
- [ ] **Streaming** - Large data is streamed rather than loaded entirely into memory where appropriate

### Memory Patterns

- [ ] **No memory leaks** - Temporary objects are freed, no circular references that prevent GC
- [ ] **Appropriate caching** - Caching is used where beneficial but with proper eviction to prevent unbounded growth
- [ ] **Object reuse** - Objects are reused where appropriate instead of repeatedly allocated

### I/O Efficiency

- [ ] **Batching** - Multiple small operations are batched into fewer larger operations where possible
- [ ] **Connection reuse** - Database and HTTP connections are reused via pools
- [ ] **Async I/O** - I/O-bound operations use async patterns to avoid blocking threads

## Documentation

### Code Documentation

- [ ] **Public API documented** - Public functions, classes, and methods have documentation
- [ ] **Parameter documentation** - Function parameters and return values are documented
- [ ] **Non-obvious behavior documented** - Edge cases, side effects, and non-obvious behavior are documented

### Change Documentation

- [ ] **Changelog updated** - Significant changes are documented in changelog if the project maintains one
- [ ] **Migration guide** - Breaking changes include migration instructions
- [ ] **README updated** - Changes to setup, configuration, or usage are reflected in README

## Language-Specific Considerations

### Python

- [ ] **Type hints** - New code includes type hints consistent with the rest of the codebase
- [ ] **Context managers** - Resources are managed with context managers (`with` statements)
- [ ] **No bare except** - Exception handlers specify exception types, not bare `except:`
- [ ] **f-strings or format** - String formatting uses f-strings or `.format()`, not `%` formatting (unless project convention)

### JavaScript/TypeScript

- [ ] **Strict mode** - TypeScript strict mode is respected; no `any` types without justification
- [ ] **Async/await** - Promises use async/await syntax consistently
- [ ] **Null handling** - Null and undefined are handled explicitly
- [ ] **No prototype pollution** - Object operations don't allow prototype pollution

### Go

- [ ] **Error handling** - Errors are checked and handled, not ignored with `_`
- [ ] **Context propagation** - Context is propagated for cancellation and timeouts
- [ ] **Resource cleanup** - Deferred cleanup is used for resources
- [ ] **No data races** - Race detector would not flag issues

### Java/Kotlin

- [ ] **Null safety** - Nullable types are handled explicitly
- [ ] **Resource management** - Try-with-resources is used for closeable resources
- [ ] **Thread safety** - Collections exposed across threads are thread-safe
- [ ] **Immutability** - Objects are immutable where possible

### Rust

- [ ] **Error handling** - Results are handled explicitly, not unwrapped without justification
- [ ] **Ownership clarity** - Ownership and borrowing are clear and idiomatic
- [ ] **No unsafe without justification** - Unsafe blocks have clear justification and are minimal

## Data Flow Integrity (delegated)

Round-trip correctness, dual-path consistency, writer-reader alignment, external state alignment, crash safety, and stored-state invalidation are the domain of the **boundary-bug-hunter** skill. Run that skill on the diff in aggressive mode and incorporate its findings into the review.

Do not duplicate boundary-bug-hunter's analysis here. Its output is more rigorous, produces a refusable gate (`boundary-status.json`), and persists explicit out-of-scope decisions to `.boundary-deferrals.json` for cross-cycle audit.

If for some reason boundary-bug-hunter cannot be run, note this in the review under "Coverage gaps in this review" and apply the items below as a fallback heuristic only — knowing they are weaker than a real seam-test gate.

## Observability & Debuggability

A change that ships with worse observability than what it replaced is a regression even if every test passes — debugging in production becomes more expensive forever after.

- [ ] **Logging at decision points** — Branches taken, retries attempted, fallbacks triggered, external-call outcomes (success / failure / timeout). A code path that silently makes a choice will be invisible in incident review.
- [ ] **Correlation propagation** — Every new log line, span, or metric within a request includes the project's correlation key (request ID, trace ID, user ID, tenant ID — whatever the codebase uses). A log line without correlation can't be joined to anything.
- [ ] **Log level appropriateness** — Errors logged at `error`/`warn`. Routine flow at `info`/`debug`. Don't log normal operation at `error` (alert fatigue) or genuine failures at `debug` (silently lost).
- [ ] **Structured logging used** — If the project uses structured logging (JSON / key-value), new log statements use the same shape. Free-form printf-style strings inside a structured-logging codebase create gaps.
- [ ] **No PII / secrets in logs** — User input, tokens, full headers, request bodies — none should land in logs without redaction. Verify by reading the new log lines as if you were a security auditor.
- [ ] **Metrics for new code paths** — New endpoints / queue handlers / cron jobs emit at least: invocation count, duration, error count. Without these, on-call cannot tell whether the new code is healthy.
- [ ] **No removed observability** — If the diff deletes log statements, metrics, or trace spans, the removal is justified (e.g., it was wrong). "Cleanup" of log lines that someone may rely on for debugging is a regression.
- [ ] **Distributed-trace continuity** — If the change adds a new internal hop (service-to-service, queue producer/consumer), trace context is propagated. Otherwise traces split into orphans at the new boundary.

## Rollout & Migration Safety

Risky changes need a way to be reversed without a re-deploy. Reviewers must verify the safety machinery exists before approving.

- [ ] **Feature-flagged when appropriate** — Significant behavior changes (new endpoints, altered request handling, new scheduled jobs) are gated behind a feature flag with a default-off state, so the change can be turned off without redeploy.
- [ ] **Kill switch exists for risky paths** — For changes that could cause cost spikes (new external API calls), data corruption (new write paths), or user-visible breakage, there is a configuration knob to disable the path immediately.
- [ ] **Migrations are reversible OR explicitly one-way** — Schema/data migrations have a `down()` that works on data created after `up()` ran, OR are explicitly documented as one-way with the rollback strategy stated.
- [ ] **No coupled deploys without explicit ordering** — If the change requires two services or a service+frontend to be deployed in a specific order, the order is documented in the PR description and in any deploy runbook. Silent ordering requirements cause outages.
- [ ] **Backwards-compatible during rollout** — During the deploy window, old code (which is still running) and new code (just deployed) coexist. Verify nothing in the diff breaks that overlap window — old readers can handle new writes, new readers can handle old writes.
- [ ] **Rollout plan in PR description** — For non-trivial changes: gradual rollout? Canary? Specific tenants first? Or "deploy and watch"? If unstated, ask.
- [ ] **Telemetry for rollout decisions** — How will the team know the rollout is succeeding? Specific metrics or dashboards exist before the rollout starts.

## Error UX & Operator Affordance

Errors are part of the product. They reach users, customer support, on-call engineers, and external API consumers. Bad error UX is a quiet but constant tax.

- [ ] **Error messages are actionable** — A new error path tells the receiver what went wrong AND what to do about it. "Invalid input" is not actionable. "Field `email` must be a valid email — got `<redacted>`" is.
- [ ] **No internal-detail leakage** — Error messages reaching external users don't include stack traces, file paths, internal hostnames, query fragments, or any infrastructure shape. Internal logs may include those; user-facing messages must not.
- [ ] **Failure modes distinct** — A 500 because the database is down should not look identical to a 500 because the request was malformed. Different failure modes need different signals so the receiver can respond differently (retry vs fix).
- [ ] **Timeouts and retries communicated** — When the system is retrying or has timed out, the receiver knows. A request that hangs with no signal is worse than a fast failure.
- [ ] **Customer-support-readable** — A support engineer looking at the error message and the correlation ID can identify the customer, the request, the failure point. If they need a full debugging session, the error UX is too thin.
- [ ] **Localization-friendly** — If the project supports multiple languages, error messages are sourced from the localization layer, not hardcoded strings. New hardcoded English strings in a localized codebase are a regression.

## Configuration & Environment

New config is a permanent operational tax. Reviewers verify it is justified, defaulted, documented, and discoverable.

- [ ] **New env vars / config keys documented** — Every new config knob is documented in the project's config reference (CLAUDE.md, README, `.env.example`, helm values, whatever the project uses). Undocumented config keys silently differ between environments.
- [ ] **Sensible defaults** — New config has a default that is safe and reasonable. Required-with-no-default config breaks startup in any environment that hasn't been updated.
- [ ] **Secrets via secrets manager** — New secrets do not live in plaintext config files, env-var examples, or commit history. They are loaded from the project's secrets manager (Vault, AWS Secrets Manager, k8s secrets, 1Password, etc.).
- [ ] **Config change is observable** — When config changes at runtime (hot-reload, dynamic config systems), the change is logged so an operator can correlate behavior shifts to config edits.
- [ ] **No environment-specific code paths via raw env checks** — `if (process.env.NODE_ENV === 'production')` scattered through business logic is a smell. Environment-specific behavior should live behind a single config interface.
- [ ] **Test/CI environments updated** — If the change adds a required config key, CI configs, dev `.env.example`, and staging configs are updated in the same PR. Otherwise CI breaks for the next person.

## Sensitive Data & Compliance

(Extends the Security section above.)

- [ ] **PII inventory honest** — If the change introduces a new field that holds PII (name, email, phone, address, IP, biometric, location, behavior data), the project's PII inventory / data-classification doc is updated. Hidden PII fields are a compliance risk.
- [ ] **Retention applies** — New data subject to retention policy is captured by the project's retention/deletion mechanism (TTL, soft-delete, periodic purge job). New eternal data is rare and should be justified.
- [ ] **Right-to-deletion supported** — In jurisdictions with deletion rights, new data has a path to be deleted on user request. Foreign-key constraints that prevent deletion are findings.
- [ ] **Audit log for security-relevant operations** — New code that grants/revokes permissions, accesses sensitive data, modifies access policies, or performs payments emits an audit-log event. Compliance and forensics depend on this.
- [ ] **Cross-region data movement disclosed** — If the change causes data to flow across regions or to a new third-party processor, the data-protection impact assessment / DPA list is updated.

## External Dependencies & Rate Limits

- [ ] **New external calls counted** — A new HTTP/gRPC/queue call to an external service adds to that service's quota. The PR description acknowledges the volume and confirms the quota is not at risk.
- [ ] **Backoff and retry budget** — New external calls implement exponential backoff with jitter and a retry budget. Unbounded retries on failure cause cascading outages.
- [ ] **Circuit breaker where appropriate** — For external dependencies on the request critical path, a circuit breaker (or the equivalent timeout/fallback pattern) prevents the dependency's outage from becoming yours.
- [ ] **Vendor failure mode handled** — When the external service returns a malformed response, an unexpected status, or times out, the code degrades gracefully. "It will never return that" is not handling.
