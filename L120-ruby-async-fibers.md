# L120-ruby-async-fibers
----
* Author(s): Claudio Raimondi
* Approver: a11r
* Status: Draft
* Implemented in: Ruby
* Last updated: [Date]
* Discussion at: <google group thread> (to be filled after thread exists)

## Abstract

This proposal suggests replacing the current thread-based concurrency model in the official Ruby gRPC implementation with the Async gem, which uses fibers for lightweight, non-blocking concurrency. This change is intended to improve performance, scalability, and resource usage for gRPC Ruby servers handling high-concurrency workloads.

## Background

Currently, the Ruby implementation of gRPC uses threads to manage concurrent requests. While threads provide basic concurrency, they come with significant overhead in terms of memory usage and context switching. With high-concurrency scenarios, thread-based concurrency can lead to bottlenecks that reduce the server's ability to handle requests efficiently.

The Async gem provides a lightweight concurrency model using fibers, which are managed by Ruby’s event loop. Fibers are more efficient than threads in terms of memory usage and context switching and can allow for better scalability without the associated overhead.

### Related Proposals:
* None

## Proposal

I propose refactoring the Ruby gRPC server to use the Async gem or directly fibers for concurrency, replacing the current thread-based model. Specifically:
- Replace the existing thread-based model with Async-based coroutines to handle incoming requests.
- Modify the gRPC server to operate within an event loop, allowing for non-blocking I/O operations.
- Ensure backward compatibility with existing Ruby applications using gRPC, while also providing clear documentation for developers transitioning to the Async model.

### Temporary environment variable protection

- `GRPC_ASYNC_CONCURRENCY_ENABLED` (default: `false`): This environment variable will control whether the Async concurrency model is enabled or not. By default, it will be set to `false` to prevent disrupting existing environments. After extensive testing, this can be toggled to `true` for production environments.

## Rationale

### Alternate Approaches:
1. **Continued use of threads:** The current threading model could remain in place. However, this would not address the underlying performance bottlenecks that are evident when scaling to high-concurrency workloads.
2. **Fibers (manual management):** Ruby fibers, which are the core of the Async gem, could be used directly without the Async gem. However, this would require more effort to manage fibers manually and would not offer the full benefits of Async's abstraction and optimizations.

### Trade-offs:
- **Async Gem vs Threads:** Async allows for better resource management and non-blocking operations. The trade-off is that developers must learn and adopt a different concurrency model, which may introduce development complexity initially.
- **Async vs Fibers Directly:** Using the Async gem abstracts the management of fibers, reducing the complexity of the implementation and providing additional features that could help with debugging and performance monitoring.

## Implementation

### Steps:
1. **Prototype:** A prototype will be developed to integrate Async into the Ruby gRPC server, replacing threads with fibers and coroutines.
2. **Benchmarking:** Performance benchmarks will be run comparing the thread-based and Async-based concurrency models to measure improvements in throughput, memory usage, and scalability.
3. **Documentation:** Extensive documentation will be written to guide developers in transitioning to the new Async-based model.
4. **Community Feedback:** The proposal will be submitted for discussion, with feedback from the community incorporated to refine the approach.
5. **Final Implementation:** After community approval, the Async concurrency model will be fully integrated into the Ruby gRPC server.

The implementation will be carried out in stages, with a focus on maintaining backward compatibility during the transition.

## Open issues

- **Compatibility with existing Ruby codebases:** Ensuring that existing Ruby gRPC applications can run without modification during the transition period.
- **Error handling and debugging:** Implementing clear error messages and debugging tools to assist developers in adapting to the new Async model.