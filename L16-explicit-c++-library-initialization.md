Explicit C++ library initialization
----
* Author(s): yang-g
* Approver: ctiller
* Status: Draft
* Implemented in: n/a
* Last updated: December 8, 2017
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/qOJaIoIzAu0

## Abstract

Add a new API to make gRPC C++ library initialization explicit.

## Background

gRPC C library has `grpc_init()` function to do various one-time initialization.
There is a `grpc_shutdown()` function to do the corresponding global cleanups.
These functions can be called multiple times as long as they are in pairs. Only
the first call to `grpc_init()` and the last call to `grpc_shutdown()` take
effect and other calls will just add or remove a reference.

In gRPC C++, we intended to hide the calls from the user. A class
`GrpcLibraryCodegen` binding the `grpc_init/grpc_shutdown` calls to its lifetime
is created. Some top level gRPC C++ classes inherit from `GrpcLibraryCodegen`
so that the C library will be initialized before the C++ level logic is started.
These classes include `Server`, `Channel`, credentials, etc. The
`GrpcLibraryCodegen` objects are created on the stack in some functions to cover
corner cases.

### Related Proposals:

N/A

## Proposal

Add a new public `GrpcLibrary` class and require gRPC C++ library users to
instantiate a `GrpcLibrary` object and keep it alive until the end of the use
of gRPC.

## Rationale

While it is convenient for the users to not need a centric initialization point
of gRPC library, as there is more integration with other libraries, the current
model seems not sustainable. We have seen some problems because of missing
coverage of `grpc_init`:

* When an error happens in creating a top-level gRPC C++ object A, a consequent
  object B making use of A may wrongly assume that `grpc_init()` has been
  called.  For example, a `GrpcLibraryCodegen` object is created on the stack in
  `CreateCustomChannel()` just in case that the passed in credentials is a
  `nullptr`.
* When creating a `ChannelCredentials`, even though the object is inherited
  from `GrpcLibraryCodegen`, we still need to create a `GrpcLibraryCodegen` to
  cover the steps right before the object is constructed.
* When a gRPC client schedules a callback in a thread not managed by gRPC, and
  the callback may race with `grpc_shutdown()` in the main thread because of
  destruction of top-level C++ objects.
* When we share memory with another library, a `grpc_slice` is handed off with a
  `grpc_slice_unref` as the destroy function. We cannot control when the final
  destruction happens and thus cannot guarantee the destruction is covered by
  `grpc_init`.
* Racing destruction of a `Server` and `ServerContext`, e.g.
  https://github.com/grpc/grpc/issues/12699
* We previously tried to stash a `GrpcLibraryCodegen` somewhere, hoping
  to fix a problem, e.g. https://github.com/grpc/grpc/pull/9136.
* We initialize the global start time in a first `grpc_init()` and if the code
  goes out of the last `grpc_shutdown()` and comes back with `grpc_init()`
  again, the global start time will be refreshed. However, we keep a global
  `google_default_credentials` object, which contains an expiration time
  relative to the start time. If the object is reused after the start time is
  refreshed, the absolute expiration time will be wrong.

In summary, because we do not have a gRPC C++ object that lives throughout the
lifetime of all gRPC internals. It is hard to guarantee that `grpc_init` is
called properly when needed, especially considering the increasing integration
between gRPC and other libraries. It is also sometimes not immediately clear
that a missing `grpc_init` is the problem when a bug is reported.

## Implementation

Make `grpc::GrpcLibrary` a first class C++ object and mandate its existence
throughout the time of gRPC usage.  With this, users do not need to worry about
the lifetime of objects and the existing sites of creation and inheritance can
be removed (incrementally).

The new class will be exposed and advertised but its use is not enforced to give
users time to migrate. Documents should be updated to require the use of it.
The current `grpc::GrpcLibraryCodegen` will detect whether a `grpc::GrpcLibrary`
has been created and will log a warning if it is not.
The `grpc::GrpcLibraryCodegen` is deprecated and removed. The previous warning may
become an assertion.

Related fixes will be needed such as for users setting polling engine later than
creating the `GrpcLibrary` object.

Note: `grpc::internal::GrpcLibrary` exists and we will need to rename it if we
want to expose a new class with the same name under `::grpc` namespace to avoid
confusion.

The advantage is that it prevents all the issues we saw above and reduces
complexity of gRPC C++ code.

The disadvantage is that since this a new API requirement, users will need to
create this object in some place. This may be a problem for library owners, e.g.
a `foo_library` may suddenly require its users to create a
`grpc::GrpcLibrary` because `foo_library` depends on gRPC (maybe transitively).

### Alternatives

* We could ask users to call `grpc_init()` and `grpc_shutdown()` on their own.
This also solves the problem but is no better than having a C++ object
to do the same (and saving one call).
* We can call `grpc_init()` in all the points leaving the gRPC library and call
`grpc_shutdown()` at the end either in or out of gRPC. This may cause per-rpc
calling of the functions and may incur significant performance impact.

