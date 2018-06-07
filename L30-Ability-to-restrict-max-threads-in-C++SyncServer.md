* Author(s): Sree Kuchibhotla (sreecha)
* Approver: vjpai
* Status: In Review
* Implemented in: C++
* Last updated: June 6, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/YQQY0pGG9MI

## Abstract
C++ `ThreadManager` is a specialized thread pool used to implement the C++ Synchronous server. 

Currently, `ThreadManager` can be configured to have minimum and maximum number of polling threads. While this helps to always have threads available to poll for work, it does nothing to throttle polling if the `ThreadManager` is overloaded with work.

The proposal here is to add a notion of 'thread quota' to the `resource_quota` object. Currently, a resource quota object can be attached to the server (via `ServerBuilder::SetResourceQuota`). The idea here is to set a maximum number of threads on that resource quota object.

Each of the `ThreadManager` objects in a given server check with the Server's resource quota object when creating new threads and will not create new threads if no quota is available.

## Details
More concretely, the following API changes are being proposed

### 1. New *public* C++ API: `ServerBuilder::SetMaxThreads`
 
```C++
// File: include/grpcpp/resource_quota.h

/// ResourceQuota represents a bound on memory and threads usage by the gRPC
/// library. A ResourceQuota can be attached to a server (via \a ServerBuilder),
/// or a client channel (via \a ChannelArguments).
///
/// gRPC will attempt to keep memory and threads used by all attached entities
/// below the ResourceQuota bound.
class ResourceQuota final : private GrpcLibraryCodegen {
 public:
  ...
  /// Set the max number of threads that can be allocated from this
  /// ResourceQuota object.
  ///
  /// If the new_max_threads value is smaller than the current value, no new
  /// threads are allocated until the number of active threads fall below
  /// new_max_threads. There is no time bound on when this may happen i.e none
  /// of the current threads are forcefully destroyed and all threads run their
  /// normal course.
  ResourceQuota& SetMaxThreads(int new_max_threads);
  ...     
```
* Max threads are set to INT_MAX by default
* In the initial implementation, I plan to divide the max_threads equally among multiple `ThreadManager` objects (there are as many `ThreadManager` objects as the number of server completion queues). It is not clear on which strategy is better at this point 
  - (1) Have all thread managers create threads from a common pool (but potentially starving some thread managers and also making the quota-check a potential global contention point)
  - (2) Divide the max_threads equally among thread managers (with the downside that some thread managers are "over provisioned" while some might be "under provisioned").
  
  I am going with option (2) for now and this may change in future. 

### 2. New *public* Core-Surface API: `grpc_resource_quota_set_max_threads`

```C++
// File: include/grpc/grpc.h

/** Update the size of the maximum number of threads allowed */
GRPCAPI void grpc_resource_quota_set_max_threads(
    grpc_resource_quota* resource_quota, int new_max_threads);

```
### 3. New *Private* Core APIs: `grpc_resource_user_alloc_threads` and `grpc_resource_user_free_threads`

This is a private API and may change. I am including this here just to give an idea of how I plan to implement this.
```C++
// File: src/core/lib/iomgr/resource_quota.h

/* Attempts to get quota (from the resource_user) to create 'thd_count' number
 * of threads. Returns true if successful (i.e the caller is now free to create
 * 'thd_count' number of threads or false if quota is not available */
bool grpc_resource_user_alloc_threads(grpc_resource_user* resource_user,
                                      int thd_count);
/* Releases 'thd_count' worth of quota back to the resource user. The quota
 * should have been previously obtained successfully by calling
 * grpc_resource_user_alloc_threads().
 *
 * Note: There need not be an exact one-to-one correspondence between
 * grpc_resource_user_alloc_threads() and grpc_resource_user_free_threads()
 * calls. The only requirement is that the number of threads allocated should
 * all be eventually released */
void grpc_resource_user_free_threads(grpc_resource_user* resource_user,
                                     int thd_count);
```
### Related Proposals:

N/A

## Rationale
Sometimes we might have to stop polling altogether if the server is overloaded with work. Currently there is no way to do that (since the minimum pollers setting always ensures some thread is polling for work). It was an oversight to not add max_threads as an option in the initial `ThreadManager` implementation

This has been one of the most requested features

## Open issues (if applicable)
