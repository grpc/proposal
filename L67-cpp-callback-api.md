C++ callback-based asynchronous API
----
* Author(s): vjpai, sheenaqotj, yang-g, zhouyihaiding
* Approver: markdroth
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/projects/12
* Last updated: March 22, 2021
* Discussion at https://groups.google.com/g/grpc-io/c/rXLdWWiosWg

## Abstract

Provide an asynchronous gRPC API for C++ in which the completion of RPC actions in the library will result in callbacks to user code,

## Background

Since its initial release, gRPC has provided two C++ APIs:

* Synchronous API
   - All RPC actions (such as unary calls, streaming reads, streaming writes, etc.) block for completion
   - Library provides a thread-pool so that each incoming server RPC executes its method handler in its own thread
* Completion-queue-based (aka CQ-based) asynchronous API
   - Application associates each RPC action that it initiates with a tag
   - The library performs each RPC action
   - The library posts the tag of a completed action onto a completion queue
   - The application must poll the completion queue to determine which asynchronously-initiated actions have completed
   - The application must provide and manage its own threads
   - Server RPCs don't have any library-invoked method handler; instead the application is responsible for executing the actions for an RPC once it is notified of an incoming RPC via the completion queue

The goal of the synchronous version is to be easy to program. However, this comes at the cost of high thread-switching overhead and high thread storage for systems with many concurrent RPCs. On the other hand, the asynchronous API allows the application full control over its threading and thus can scale further. The biggest problem with the asynchronous API is that it is just difficult to use. Server RPCs must be explicitly requested, RPC polling must be explicitly controlled by the application, lifetime management is complicated, etc. These have proved sufficiently difficult that the full features of the asynchronous API are basically never used by applications. Even if one can use the async API correctly, it also presents challenges in deciding how many completion queues to use and how many threads to use for polling them, as one can either optimize for reducing thread hops, avoiding stranding, reducing CQ contention, or improving locality. These goals are often in conflict and require substantial tuning.

### Related proposals

* The C++ callback API has an implementation that is built on top of a new [callback completion queue in core](https://github.com/grpc/proposal/pull/181). There is also another implementation, discussed [below](#implementation).
* The API structure has substantial similarities to the gRPC-Node and gRPC-Java APIs.

## Proposal

The callback API is designed to have the performance and thread scalability of an asynchronous API without the burdensome programming model of the completion-queue-based model. In particular, the following are fundamental guiding principles of the API:

* Library directly calls user-specified code at the completion of RPC actions. This user code is run from the library's own threads, so it is very important that it must not wait for completion of any blocking operations (e.g., condition variable waits, invoking synchronous RPCs, blocking file I/O).
* No explicit polling required for notification of completion of RPC actions.
   - In practice, these requirements mean that there must be a library-controlled poller for monitoring such actions. This is discussed in more detail in the [Implementation](#implementation) section below.
* As in the synchronous API, server RPCs have an application-defined method handler function as part of their service definition. The library invokes this method handler when a new server RPC starts.
* Like the synchronous API and unlike the completion-queue-based asynchronous API, there is no need for the application to "request" new server RPCs. Server RPC context structures will be allocated and have their resources allocated as and when RPCs arrive at the server.

### Reactor model

The most general form of the callback API is built around a _reactor_ model. Each type of RPC has a reactor base class provided by the library. These types are:

* `ClientUnaryReactor` and `ServerUnaryReactor` for unary RPCs
* `ClientBidiReactor` and `ServerBidiReactor` for bidi-streaming RPCs
* `ClientReadReactor` and `ServerWriteReactor` for server-streaming RPCs
* `ClientWriteReactor` and `ServerReadReactor` for client-streaming RPCs

Client RPC invocations from a stub provide a reactor pointer as one of their arguments, and the method handler of a server RPC must return a reactor pointer.

These base classes provide three types of methods:

1. Operation-initiation methods: start an asynchronous activity in the RPC. These are methods provided by the class and are not virtual. These are invoked by the application logic. All of these have a `void` return type. The `ReadMessageType` below is the request type for a server RPC and the response type for a client RPC; the `WriteMessageType` is the response type for a server RPC or the request type for a client RPC.
   - `void StartCall()`: (*Client only*) Initiates the operations of a call from the client, including sending any client-side initial metadata associated with the RPC. Must be called exactly once. No reads or writes will actually be started until this is called (i.e., any previous calls to `StartRead`, `StartWrite`, or `StartWritesDone` will be queued until `StartCall` is invoked). This operation is not needed at the server side since streaming operations at the server are released from backlog automatically by the library as soon as the application returns a reactor from the method handler, and because there is a separate method just for sending initial metadata.
   - `void StartSendInitialMetadata()`: (*Server only*) Sends server-side initial metadata. To be used in cases where initial metadata should be sent without sending a message. Optional; if not called, initial metadata will be sent when `StartWrite` or `Finish` is called. May not be invoked more than once or after `StartWrite` or `Finish` has been called. This does not exist at the client because sending initial metadata is part of `StartCall`.
   - `void StartRead(ReadMessageType*)`: Starts a read of a message into the object pointed to by the argument. `OnReadDone` will be invoked when the read is complete. Only one read may be outstanding at any given time for an RPC (though a read and a write can be concurrent with each other). If this operation is invoked by a client before calling `StartCall` or by a server before returning from the method handler, it will be queued until one of those events happens and will not actually trigger any activity or reactions until it is thereby released from the queue.
   - `void StartWrite(const WriteMessageType*)`: Starts a write of the object pointed to by the argument. `OnWriteDone` will be invoked when the write is complete. Only one write may be outstanding at any given time for an RPC (though a read and a write can be concurrent with each other).  As with `StartRead`, if this operation is invoked by a client before calling `StartCall` or by a server before returning from the method handler, it will be queued until one of those events happens and will not actually trigger any activity or reactions until it is thereby released from the queue.
   - `void StartWritesDone()`: (*Client only*) For client RPCs to indicate that there are no more writes coming in this stream.  `OnWritesDoneDone` will be invoked when this operation is complete. This causes future read operations on the server RPC to indicate that there is no more data available. Highly recommended but technically optional; may not be called more than once per call.  As with `StartRead` and `StartWrite`, if this operation is invoked by a client before calling `StartCall` or by a server before returning from the method handler, it will be queued until one of those events happens and will not actually trigger any activity or reactions until it is thereby released from the queue.
   - `void Finish(Status)`: (*Server only*) Sends completion status to the client, asynchronously. Must be called exactly once for all server RPCs, even for those that have already been cancelled. No further operation-initiation methods may be invoked after `Finish`.
1. Operation-completion reaction methods: notification of completion of asynchronous RPC activity. These are all virtual methods that default to an empty function (i.e., `{}`) but may be overridden by the application's reactor definition. These are invoked by the library. All of these have a `void` return type. Most take a `bool ok` argument to indicate whether the operation completed "normally," as explained below.
   - `void OnReadInitialMetadataDone(bool ok)`: (*Client only*) Invoked by the library to notify that the server has sent an initial metadata response to a client RPC. If `ok` is true, then the RPC received initial metadata normally. If it is false, there is no initial metadata either because the call has failed or because the call received a trailers-only response (which means that there was no actual message and that any information normally sent in initial metadata has been dispatched instead to trailing metadata, which is allowed in the gRPC HTTP/2 transport protocol). This reaction is automatically invoked by the library for RPCs of all varieties; it is uncommonly used as an application-defined reaction however.
   - `void OnReadDone(bool ok)`: Invoked by the library in response to a `StartRead` operation. The `ok` argument indicates whether a message was read as expected. A false `ok` could mean a failed RPC (e.g., cancellation) or a case where no data is possible because the other side has already ended its writes (e.g., seen at the server-side after the client has called `StartWritesDone`).
   - `void OnWriteDone(bool ok)`: Invoked by the library in response to a `StartWrite` operation. The `ok` argument that indicates whether the write was successfully sent; a false value indicates an RPC failure.
   - `void OnWritesDoneDone(bool ok)`: (*Client only*) Invoked by the library in response to a `StartWritesDone` operation. The bool `ok` argument that indicates whether the writes-done operation was successfully completed; a false value indicates an RPC failure.
   - `void OnCancel()`: (*Server only*) Invoked by the library if an RPC is canceled before it has a chance to successfully send status to the client side. The reaction may be used for any cleanup associated with cancellation or to guide the behavior of other parts of the system (e.g., by setting a flag in the service logic associated with this RPC to stop further processing since the RPC won't be able to send outbound data anyway). Note that servers must call `Finish` even for RPCs that have already been canceled as this is required to cleanup all their library state and move them to a state that allows for calling `OnDone`.
   - `void OnDone(const Status&)` at the client, `void OnDone()` at the server: Invoked by the library when all outstanding and required RPC operations are completed for a given RPC. For the client-side, it additionally provides the status of the RPC (either as sent by the server with its `Finish` call or as provided by the library to indicate a failure), in which case the signature is `void OnDone(const Status&)`. The server version has no argument, and thus has a signature of `void OnDone()`. Should be used for any application-level RPC-specific cleanup.
   - _Thread safety_: the above calls may take place concurrently, except that `OnDone` will always take place after all other reactions. No further RPC operations are permitted to be issued after `OnDone` is invoked.
   - **IMPORTANT USAGE NOTE** : code in any reaction must not block for an arbitrary amount of time since reactions are executed on a finite-sized, library-controlled threadpool. If any long-term blocking operations (like sleeps, file I/O, synchronous RPCs, or waiting on a condition variable) must be invoked as part of the application logic, then it is important to push that outside the reaction so that the reaction can complete in a timely fashion. One way of doing this is to push that code to a separate application-controlled thread.
1. RPC completion-prevention methods. These are methods provided by the class and are not virtual. They are only present at the client-side because the completion of a server RPC is clearly requested when the application invokes `Finish`. These methods are invoked by the application logic. All of these have a `void` return type.
   - `void AddHold()`: (*Client only*) This prevents the RPC from being considered complete (ready for `OnDone`) until each `AddHold` on an RPC's reactor is matched to a corresponding `RemoveHold`. An application uses this operation before it performs any  _extra-reaction flows_, which refers to streaming operations initiated from outside a reaction method. Note that an RPC cannot complete before `StartCall`, so holds are not needed for any extra-reaction flows that take place before `StartCall`. As long as there are any holds present on an RPC, though, it may not have `OnDone` called on it, even if it has already received server status and has no other operations outstanding. May be called 0 or more times on any client RPC.
   - `void AddMultipleHolds(int holds)`: (*Client only*) Shorthand for `holds` invocations of `AddHold` .
   - `void RemoveHold()`: (*Client only*) Removes a hold reference on this client RPC. Must be called exactly as many times as `AddHold` was called on the RPC, and may not be called more times than `AddHold` has been called so far for any RPC. Once all holds have been removed, the server has provided status, and all outstanding or required operations have completed for an RPC, the library will invoke `OnDone` for that RPC.

Examples are provided in [the PR to de-experimentalize the callback API](https://github.com/grpc/grpc/pull/25728).

### Unary RPC shortcuts

As a shortcut, client-side unary RPCs _may_ bypass the reactor model by directly providing a `std::function` for the library to call at completion rather than a reactor object pointer. This is passed as the final argument to the stub call, just as the reactor would be in the more general case. This is semantically equivalent to a reactor in which the `OnDone` function simply invokes the specified function (but can be implemented in a slightly faster way since such an RPC will definitely not wait separately for initial metadata from the server) and all other reactions are left empty. In practice, this is the common and recommended model for client-side unary RPCs, unless they have a specific need to wait for initial metadata before getting their full response message. As in the reactor model, the function provided as a callback may not include operations that block for an arbitrary amount of time.

Server-side unary RPCs have the option of returning a library-provided default reactor when their method handler is invoked. This is provided by calling [`DefaultReactor` on the `CallbackServerContext`](#servercontext-extensions). This default reactor provides a `Finish` method, but does not provide a user callback for `OnCancel` and `OnDone`. In practice, this is the common and recommended model for most server-side unary RPCs unless they specifically need to react to an `OnCancel` callback or do cleanup work after the RPC fully completes.

### ServerContext extensions

`ServerContext` is now made a derived class of `ServerContextBase`. There is a new derived class of `ServerContextBase` called `CallbackServerContext` which provides a few additional functions:

* `ServerUnaryReactor* DefaultReactor()` may be used by a method handler to return a default reactor from a unary RPC.
* `RpcAllocatorState* GetRpcAllocatorState`: see advanced topics section

Additionally, the `AsyncNotifyWhenDone` function is not present in the `CallbackServerContext`.

All method handler functions for the callback API take a `CallbackServerContext*` as their first argument. `ServerContext` (used for the sync and CQ-based async APIs) and `CallbackServerContext` (used for the callback API) actually use the same underlying structure and thus their object pointers are meaningfully convertible to each other via a `static_cast` to `ServerContextBase*`. We recommend that any helper functions that need to work across API variants should use a `ServerContextBase` pointer or reference as their argument rather than a `ServerContext` or `CallbackServerContext` pointer or reference. For example, `ClientContext::FromServerContext` now uses a `ServerContextBase*` as its argument; this is not a breaking API change since the argument is now a parent class of the previous argument's class.

### Advanced topics

#### Application-managed server memory allocation

Callback services must allocate an object for the `CallbackServerContext` and for the request and response objects of a unary call.  Applications can supply a per-method custom memory allocator for gRPC server to use to allocate and deallocate the request and response messages, as well as a per-server custom memory allocator for context objects. These can be used for purposes like early or delayed release, freelist-based allocation, or arena-based allocation. For each unary RPC method, there is a generated method in the server called `SetMessageAllocatorFor_*MethodName*` . For each server, there is a method called `SetContextAllocator`. Each of these has numerous classes involved, and the best examples for how to use these features lives in the gRPC tests directory.

* [Message allocator usage example](https://github.com/grpc/grpc/blob/master/test/cpp/end2end/message_allocator_end2end_test.cc)
* [Context allocator usage example](https://github.com/grpc/grpc/blob/master/test/cpp/end2end/context_allocator_end2end_test.cc)

#### Generic (non-code-generated) services

`RegisterCallbackGenericService` is a new method of `ServerBuilder` to allow for processing of generic (unparsed) RPCs. This is similar to the pre-existing `RegisterAsyncGenericService` but uses the callback API and reactors rather than the CQ-based async API. It is expected to be used primarily for generic gRPC proxies where the exact serialization format or list of supported methods is unknown.

#### Per-method specification

Just as with async services, callback services may also be specified on a method-by-method basis (using the syntax `WithCallbackMethod_*MethodName*`), with any unlisted methods being treated as sync RPCs. The shorthand `CallbackService` declares every method as being processed by the callback API. For example:

* `Foo::Service` -- purely synchronous service
* `Foo::CallbackService` -- purely callback service
* `Foo::WithCallbackMethod_Bar<Service>` -- synchronous service except for callback method `Bar`
* `Foo::WithCallbackMethod_Bar<WithCallbackMethod_Baz<Service>>` -- synchronous service except for callback methods `Bar` and `Baz`

## Rationale

Besides the content described in the background section, the rationale also includes early and consistent user demand for this feature as well as the fact that many users were simply spinning up a callback model on top of gRPC's completion queue-based asynchronous model.

## Implementation

There is more than one mechanism available for implementing the background polling required by the C++ callback API. One has been implemented [on top of the C++ completion queue API](https://github.com/grpc/grpc/pull/25169). In this approach, the callback API uses a number of library-owned threads to call `Next` on an async CQ that is owned by the internal implementation. Currently, the thread count is automatically selected by the library with no user input and is set to half the system's core count, but no less than 2 and no more than 16. This selection is subject to change in the future based on our team's ongoing performance analysis and tuning efforts. Despite being built on the CQ-based async API, the developer using the callback API does not need to consider any of the CQ details (e.g., shutdown, polling, or even the existence of a CQ).

It is the gRPC team's intention that that implementation is only a temporary solution. A new structure called an `EventEngine` is being developed to provide the background threads needed for polling, and this sytem is also intended to provide a direct API for application use. This event engine would also allow the direct use of the [core callback API](https://github.com/grpc/proposal/blob/master/L68-core-callback-api.md) that is currently only used by the Python async implementation. If this solution is adopted, there will be a new gRFC for it. This new implementation will not change the callback API at all but rather will only affect its performance. The C++ code for the callback API already has `if` branches in place to support the use of a poller that directly supplies the background threads, so the callback API will naturally layer on top of the `EventEngine` without further development effort.

## Open issues (if applicable)

N/A. The gRPC C++ callback API has been used internally at Google for two years now, and the code and API have evolved substantially during that period.
