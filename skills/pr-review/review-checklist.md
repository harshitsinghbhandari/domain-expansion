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

## Data Flow Integrity

This section catches bugs invisible to static code analysis — where code looks correct in isolation but data is lost or corrupted as it flows through the system. These are the bugs that pass CI and break in production.

### Round-Trip Correctness

For every data value that is persisted (written to disk, database, config, API response):

- [ ] **Type vs runtime validator alignment** — If the project uses both compile-time types (TypeScript `type`/`interface`, Python type hints) AND a runtime validator (Zod, Joi, Pydantic, JSON Schema), diff the two. Every value in the compile-time type MUST exist in the runtime validator schema. Missing values are silently stripped on parse. Search for `z.object`, `z.enum`, `z.union`, `BaseModel`, `Schema` and compare against the corresponding TypeScript types or Python type hints.

- [ ] **Write → parse → re-write round-trip** — Trace a value through: code writes it → serializer stores it → parser reads it back → code uses it. Does the value survive? Check: are there fields that are written but not in the parse schema? Are there parse schemas that add defaults the writer doesn't set?

- [ ] **Sanitization strips data needed later** — If a sanitizer/normalizer processes raw input before storage, verify it doesn't strip fields that downstream code relies on. "Preserved until X" comments are a red flag — verify the preservation actually works end-to-end.

### Dual-Path Consistency

- [ ] **Duplicate logic audit** — Search for cases where the same conceptual operation exists in multiple files (e.g., status derivation, path resolution, validation). For each pair, verify they produce identical behavior for identical inputs. If they differ, determine which is canonical and flag the other.

- [ ] **Launch vs restore, create vs update symmetry** — If a resource has a "create/launch" path and an "update/restore" path, compare their configurations. Restore paths often forget flags, permissions, or setup that the create path includes. Classic pattern: create does X, Y, Z but restore only does X, Y.

- [ ] **Migration forward vs rollback symmetry** — If migration adds/transforms data, verify rollback correctly reverses it. Check: does rollback account for data created AFTER migration (new sessions, new archives)? Does it count everything that would be destroyed?

### Counter/Flag Writer-Reader Alignment

- [ ] **Writer-reader mismatch** — For every `if (counter > 0)` or `if (flag)`, trace: who increments/sets this? Are there OTHER code paths that should also set it but don't? Writer-reader mismatches are invisible when you look at either side in isolation.

### External Tool Contract Alignment

When code manages state that an external tool also tracks:

- [ ] **Moved resources need repair** — If code moves a directory that an external tool tracks (git worktree, Docker volume, tmux session dir), verify the tool's internal references are updated. For git worktrees: `git worktree repair`. For tmux: `tmux rename-session`. Don't just move the filesystem entry — update the tool's state too.

- [ ] **Renamed identifiers need propagation** — If a session name, container name, or worktree path changes, trace everywhere that identifier is used: metadata files, config references, log files, monitoring systems. All must be updated.

- [ ] **Tool-specific post-move validation** — After moving/renaming an external tool's resource, verify the tool still works with it. Run a lightweight validation command (e.g., `git status`, `tmux list-sessions`) rather than assuming the move succeeded from the filesystem perspective.

### Crash Safety

- [ ] **Partial operation recovery** — If a function performs N sequential filesystem/network operations, verify what happens if it crashes after step K. Steps 1..K are done, K+1..N are not. Is the state recoverable? Can the operation be safely re-run? Look for: marker files, idempotent operations, or cleanup of partial state.

- [ ] **Idempotency of re-run** — If an operation is interrupted and re-run, does it produce the correct result? Check: do early steps have "skip if already done" guards? Do write operations overwrite (safe) or append (unsafe on re-run)?

- [ ] **Temp file cleanup on failure** — If a write-then-rename pattern is used (atomic write), verify the temp file is cleaned up if the rename fails. Otherwise temp files accumulate indefinitely.

### Stored State Invalidation

- [ ] **Stored state vs current truth** — For any stored/cached state, ask: what happens if the underlying truth changes after the state was stored? Does the stored state become stale? Is there a mechanism to re-derive or invalidate it? Examples: PR state stored in metadata after PR is merged; cached permissions after role change; derived status after lifecycle state changes.

- [ ] **Post-action state consistency** — After a destructive or transformative action (kill, archive, migrate, restore), verify the state stored in metadata matches the new reality. Check: are there fields from the pre-action state that would cause incorrect behavior if read post-action?

- [ ] **Archive is not deletion** — When code counts or scans resources, does it treat "archived" the same as "doesn't exist"? Archived resources are invisible to scans but still exist and may contain data the user expects to keep. Verify destructive operations count archives.
