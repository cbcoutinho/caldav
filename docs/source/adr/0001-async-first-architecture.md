# ADR 0001: Async-First Architecture with Sync Wrapper

## Status

**Proposed** - For caldav v3.0

## Context

The caldav library currently maintains two parallel implementations: a synchronous API (`DAVObject`, `Calendar`, `Event`, etc.) and an asynchronous API (`AsyncDAVObject`, `AsyncCalendar`, `AsyncEvent`, etc.). While this dual approach provides flexibility for users operating in both synchronous and asynchronous contexts, it introduces significant technical debt and maintenance challenges.

### Current Problems

The current architecture suffers from approximately 1,500 lines of duplicated logic across sync and async implementations. This duplication creates a substantial maintenance burden, as every bug fix and new feature must be implemented twice, effectively doubling the work required for library evolution. The parallel maintenance requirement introduces consistency risks, where changes can inadvertently diverge between the two implementations, leading to subtle behavioral differences. Testing overhead is similarly doubled, as complete test coverage requires exercising both implementations independently. Furthermore, API evolution becomes complicated as new features must be carefully designed to work correctly in both synchronous and asynchronous paradigms.

### Previous Refactoring Attempt

A recent attempt to reduce duplication through shared utility modules (`dav_core.py` and `ical_logic.py`) was rejected during code review. The approach created what the reviewer termed "divergent implementations" - establishing two different code paths to achieve identical results. This pattern proved to be worse than honest code duplication, as it obscured the parallel nature of the implementations while adding indirection and complexity. The rejected refactoring highlighted that superficial code sharing does not address the fundamental architectural problem.

### Industry Best Practices

Modern Python HTTP libraries have converged on async-first architecture as the solution to this problem. The httpx library maintains asynchronous code as the primary implementation and uses the unasync tool to automatically generate the synchronous version. The asyncpg library provides its core as async with sync wrappers available through controlled use of `asyncio.run()`. The aiohttp library takes a pure async approach, leaving synchronous wrapping to users when needed. The pattern across the ecosystem is clear: write asynchronous code once and derive synchronous variants automatically, rather than maintaining parallel implementations.

## Decision

We will adopt an async-first architecture for caldav v3.0. All core logic will be written as asynchronous methods, establishing async as the primary implementation. The synchronous API will be generated automatically through code generation or provided via thin wrappers, ensuring only async code requires active maintenance. This establishes a single source of truth for the library's business logic. As this represents a significant architectural change and may affect user code, it requires a major version bump to v3.0.

## Implementation Strategy

### Code Generation Approach

The recommended approach employs code generation similar to the pattern used by httpx and the unasync library. In this model, the source code in `caldav/_async/client.py` contains the authoritative async implementation. A code generation tool mechanically transforms this async code into synchronous equivalents in `caldav/_sync/client.py`, replacing `async def` with `def`, `await` with direct calls, and `async with` with standard context managers.

Three tooling options exist for this transformation. The unasync library represents a proven solution already used successfully by httpx, trio, and anyio. Alternatively, a custom generator could be developed and tailored specifically to caldav's needs and patterns. The generated code should be committed to the repository rather than generated on-the-fly, allowing developers to inspect, test, and debug the synchronous implementation directly.

### Public API Structure

The public API will be reorganized to clarify the distinction between sync and async interfaces. The default import path `caldav` will continue to expose the synchronous API (`DAVClient`, `Calendar`, `Event`, `Todo`, `Principal`), ensuring backward compatibility for existing users. The asynchronous API will be accessible through a dedicated `caldav.aio` module, providing a cleaner and more conventional import structure than the current `async_davclient` and `async_collection` modules.

### Migration Path

For users upgrading from v2.x to v3.0, the migration experience will differ based on their current usage pattern. Synchronous users will experience no breaking changes in their code, as the import paths and API signatures remain identical between v2.x and v3.x. Asynchronous users will benefit from cleaner import statements, consolidating their imports from scattered `async_davclient` and `async_collection` modules into the unified `caldav.aio` namespace. While this represents a breaking change for async users, the migration is straightforward and improves code clarity.

## Consequences

### Positive Outcomes

The async-first architecture establishes a single source of truth for all business logic. Bug fixes and feature implementations in the async codebase automatically propagate to the sync implementation through code generation, eliminating the risk of divergence. Maintenance burden is reduced by approximately 50%, as developers only need to maintain the async implementation. The generated nature of the sync code provides an absolute consistency guarantee - the two implementations cannot diverge because one is mechanically derived from the other.

This architecture aligns with modern Python ecosystem trends. Python 3.7 and later versions treat async as a first-class feature, and the broader ecosystem increasingly adopts async-first patterns for I/O-bound libraries. Testing becomes more efficient, as comprehensive testing of the async implementation provides confidence in both APIs, with only generation-specific edge cases requiring sync-specific tests. The approach is future-proof, positioning caldav to take advantage of async ecosystem improvements as they emerge.

### Negative Outcomes

This proposal represents a breaking change requiring a major version bump to v3.0. The implementation effort is substantial, estimated at 3-5 months of focused development affecting approximately 30 files across the codebase. The build process gains additional complexity through the introduction of a code generation step, which must be integrated into the development workflow and continuous integration pipeline.

The debugging experience may become more challenging for sync users, as the code they execute is generated rather than hand-written. While the generated code will be committed to the repository and fully inspectable, stack traces and debugging sessions will reference generated code that cannot be directly edited. Contributors face a learning curve as they must write async-first code, potentially creating a barrier for developers unfamiliar with asynchronous Python programming.

### Risks and Mitigations

Several risks must be addressed during implementation. Generated code may contain bugs not present in hand-written implementations, which we mitigate by committing generated code to the repository for thorough review and testing. Synchronous performance might degrade compared to hand-optimized implementations, requiring careful benchmarking before and after the migration to ensure code generation preserves performance characteristics. Breaking user code during the v2.x to v3.0 transition is inevitable for async users, which we address through comprehensive migration guides and deprecation warnings in the final v2.x release. Complex async patterns may not translate cleanly to synchronous code, requiring careful API design to ensure the async implementation uses generation-friendly patterns.

## Alternatives Considered

### Alternative 1: Async Inherits from Sync

One alternative would have the `AsyncDAVClient` class inherit from `DAVClient`, allowing async methods to override sync methods while calling `super()` for shared logic. This approach was rejected because it still requires maintaining two complete implementations, merely changing how they relate rather than eliminating duplication. The inheritance relationship creates coupling between sync and async implementations, increasing complexity and making both harder to understand. Most importantly, it does not meaningfully reduce code duplication, as the async implementation must still be written and maintained independently.

### Alternative 2: Sync Wrapper via asyncio.run()

Another option would implement the sync API as a thin wrapper around the async API, using `asyncio.run()` to execute async methods synchronously. Each sync method would delegate to its async counterpart through `asyncio.run()`, eliminating code duplication entirely. This approach was rejected due to significant runtime performance concerns. The `asyncio.run()` function creates a new event loop for each invocation, introducing substantial overhead for each operation. The pattern fails entirely if the user already has an event loop running in their thread, a common scenario in modern Python applications. Error handling and resource cleanup become complicated across the sync-async boundary. While this pattern may be useful as a user-land solution for specific use cases, embedding it in the library's core architecture would create more problems than it solves.

### Alternative 3: Keep Parallel Implementations

The final alternative would maintain the current parallel implementation strategy, accepting the code duplication as a necessary cost of supporting both paradigms. This option was rejected because the maintenance burden only grows over time as the library evolves and new features are added. Consistency issues between implementations are already observable in the current codebase and will continue to emerge. The approach is fundamentally unsustainable for a library expecting long-term maintenance and evolution.

## Implementation Checklist

The implementation will proceed through five distinct phases over an estimated 4-6 month period.

**Research Phase** (2-3 weeks):
- Evaluate unasync versus custom generator versus other build tools
- Prototype code generation with 2-3 core classes to validate approach
- Benchmark sync performance comparing generated versus hand-written implementations
- Design async-friendly API patterns that generate clean synchronous code

**Core Migration** (6-8 weeks):
- Reorganize codebase structure establishing `caldav/_async/` and `caldav/_sync/` directories
- Migrate `AsyncDAVClient` as the primary authoritative implementation
- Generate `DAVClient` via selected tooling and validate correctness
- Migrate `AsyncDAVObject` and all subclasses to new structure
- Set up CI/CD pipeline integration for code generation

**Feature Parity** (4-6 weeks):
- Migrate all calendar operations to async-first implementation
- Migrate all principal and collection operations
- Migrate all object resources including Event, Todo, and Journal
- Migrate utility functions and helper modules

**Testing & Quality** (4-6 weeks):
- Port existing test suite to async-first approach
- Add integration tests exercising both sync and async APIs
- Comprehensive performance benchmarking against v2.x baseline
- Update all documentation to reflect new architecture

**Release Preparation** (2-3 weeks):
- Write comprehensive migration guide for v2.x to v3.0 upgrade path
- Add deprecation warnings in final v2.x release
- Update all examples and tutorials to demonstrate new patterns
- Conduct beta release and gather community feedback

## References

The unasync library provides async-to-sync code generation used by multiple major Python projects (https://github.com/python-trio/unasync). The httpx library demonstrates async-first architecture using unasync in production (https://github.com/encode/httpx). PEP 492 defines coroutines with async and await syntax, establishing the foundation for modern async Python (https://peps.python.org/pep-0492/). The Trio project's design principles articulate the philosophy and benefits of async-first architecture (https://trio.readthedocs.io/en/stable/design.html).

## Decision Makers

This ADR is proposed by @cbcoutinho and requires review by @tobixen as project maintainer. Community input will be solicited through the caldav-discuss mailing list and GitHub discussions before final acceptance.

## Changelog

- 2025-01-09: Initial draft (proposed status)
