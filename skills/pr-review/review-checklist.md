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
