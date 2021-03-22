C++ callback-based asynchronous API
----
* Author(s): vjpai, sheenaqotj, yang-g, zhouyihaiding
* Approver: markdroth
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/projects/12
* Last updated: March 19, 2021
* Discussion at

## Abstract

Provide an asynchronous gRPC API for C++ in which the completion of RPC actions in the library will result in callbacks to user code,

## Background

Since its initial release, gRPC has provided two C++ APIs:

* Synchronous API
   - All RPC actions (such as unary calls, streaming reads, streaming writes, etc.) block for completion
* Completion-queue-based asynchronous API
   - Application associates each RPC action that it initiates with a tag
   - The library performs each RPC action
   - The library posts the tag of a completed action onto a completion queue
   - The application must poll the completion queue to determine which asynchronously-initiated actions have completed

The goal of the synchronous version is to be easy to program. However, this comes at the cost of high thread-switching overhead and high thread storage for systems with many concurrent RPCs. On the other hand, the asynchronous API allows the application full control over its threading and thus can scale further. The biggest problem with the asynchronous API is that it is just difficult to use. Server RPCs must be explicitly requested, RPC polling must be explicitly controlled by the application, lifetime management is complicated, etc. These have proved sufficiently difficult that the full features of the asynchronous API are basically never used by applications. Even if one can use the async API correctly, it also presents challenges in deciding how many completion queues to use and how many threads to use for polling them, as one can either optimize for reducing thread hops, avoiding stranding, reducing CQ contention, or improving locality. These goals are often in conflict and require substantial tuning.

### Related proposals

* The C++ callback API is built on top of a new [callback completion queue in core](https://github.com/grpc/proposal/pull/181)
* The API structure has substantial similarities to the gRPC-Node and gRPC-Java APIs

## Proposal

The callback API is designed to have the performance and thread scalability of an asynchronous API without the burdensome programming model of the completion-queue-based model. In particular, the following are fundamental guiding principles of the API:

* Library directly calls user-specified code at the completion of RPC actions.
* No explicit polling required for notification of completion of RPC actions.
   - In practice, these requirements mean that there must be a library-controlled poller for monitoring such actions.
* As in the synchronous API, server RPCs have an application-defined method handler function as part of their service definition. The library invokes this method handler when a new server RPC starts.
* Like the synchronous API and unlike the completion-queue-based asynchronous API, there is no need for the application to "request" new server RPCs. Server RPC context structures will be allocated and have their resources allocated as and when RPCs arrive at the server.

### Reactor model

The most general form of the callback API is built around a _reactor_ model. Each type of RPC has a reactor base class provided by the library. These types are:

* `ClientUnaryReactor` for unary calls
* `ClientBidiReactor` for bidi-streaming 
* `ClientReadReactor` for server-streaming 
* `ClientWriteReactor` for client-streaming 
* `ServerUnaryReactor` for unary calls
* `ServerBidiReactor` for bidi-streaming 
* `ServerReadReactor` for client-streaming 
* `ServerWriteReactor` for server-streaming 

Client RPC invocations from a stub provide a reactor pointer as one of their arguments, and the method handler of a server RPC must return a reactor pointer.

These base classes provide three types of methods:

1. Operation-initiation methods: start an asynchronous activity in the RPC. These are methods provided by the class and are not virtual.
   - `StartRead`, `StartWrite`, `StartWritesDone`: streaming operations as appropriate to the RPC variety. *Client and server*.
   - `StartSendInitialMetadata`: for server responses where the application wants to send the metadata separately from the first response message (uncommonly used). *Server only*.
   - `Finish`: for server RPCs to provide completion status. This initiates the asynchronous transmission of the status to the client. Must be used for all server RPCs, including those that have already been cancelled. *Server only*.
1. Operation-completion reaction methods: notification of completion of asynchronous RPC activity. These are all virtual methods that default to an empty function (i.e., `{}`) but may be overridden by the application's reactor definition.
   - `OnReadDone`, `OnWriteDone`, `OnWritesDoneDone`: methods invoked by the library at the completion of an explicitly-specified streaming RPC operation. Provides a `bool ok` argument to indicate whether the completion was successful. *Client and server*. 
   - `OnReadInitialMetadataDone`: notifying client RPCs that the server has sent initial metadata. It has a bool `ok` argument to indicate whether the completion was successful. This is automatically invoked by the library for all RPCs; it is uncommonly used however. *Client only*
   - `OnCancel`: for server RPCs only, called if an RPC is canceled before it has a chance to successfully send status to the client side. Should be used for any cleanup associated with cancellation. Note that even cancelled RPCs _must_ get a `Finish` call to cleanup all their library state. *Server only*
   - `OnDone`: called when all outstanding RPC operations are completed for both client and server RPCs. For the client-side, it additionally provides the status that the server sent with its `Finish` call. Should be used for any application-level RPC-specific cleanup. *Client and server.*
   - **IMPORTANT USAGE NOTE** : code in any reaction must not block for an arbitrary amount of time since reactions are executed on a finite-sized library-controlled threadpool. If any long-term blocking operations (like sleeps, file I/O, synchronous RPCs, or waiting on a condition variable) must be invoked as part of the application logic, then it is important to push that outside the reaction so that the reaction can complete in a timely fashion. One way of doing this is to push that code to a separate application-controlled thread.
1. RPC completion prevention methods. These are methods provided by the class and are not virtual.
   - `AddHold`, `RemoveHold`: for client RPCs, prevents the RPC from being considered complete (ready for `OnDone`) until each `AddHold` is matched to a corresponding `RemoveHold`. These are used to perform _extra-reaction flows_, which refers to  streaming operations initiated from outside a reaction method or method handler. *Client only*. (These are not needed at the server because even if there are extra-reaction flows, the end of the application's involvement with a server RPC is very clearly documented by the invocation of the `Finish` call.)

Examples are provided in [the PR to de-experimentalize the callback API](https://github.com/grpc/grpc/pull/25728).

### Unary RPC shortcuts

As a shortcut, client-side unary RPCs _may_ bypass the reactor model by directly providing a `std::function` for the library to call at completion rather than a reactor object pointer. This is semantically equivalent to a reactor in which the `OnDone` function simply invokes the specified function (but can be implemented in a slightly faster way since such an RPC will definitely not wait separately for initial metadata from the server). In practice, this is the common and recommended model for client-side unary RPCs, unless they have a specific need to wait for initial metadata before getting their full response message. As in the reactor model, the function provided as a callback may not include operations that block for an arbitrary amount of time.

Server-side unary RPCs have the option of returning a library-provided default reactor when their method handler is invoked. This default reactor provides a `Finish` method, but does not provide a user callback for `OnCancel` and `OnDone`. In practice, this is the common and recommended model for most server-side unary RPCs unless they specifically need to react to an `OnCancel` callback or do cleanup work after the RPC fully completes.

### ServerContext extensions

`ServerContext` is now made a derived class of `ServerContextBase`. There is a new derived class of `ServerContextBase` called `CallbackServerContext` which provides a few additional functions:

* `DefaultReactor` may be used by a method handler to return a default reactor from a unary RPC.
* `GetRpcAllocatorState`: see advanced topics section

Additionally, the `AsyncNotifyWhenDone` function is not present in the `CallbackServerContext`. All method handler functions for the callback API take a `CallbackServerContext*` as their first argument. `ServerContext` (used for the sync and CQ-based async APIs) and `CallbackServerContext` (used for the callback API) actually use the same underlying structure and thus their object pointers are meaningfully convertible to each other via a `static_cast` to `ServerContextBase*`. We recommend that any helper functions that need to work across API variants should use a `ServerContextBase` pointer or reference as their argument rather than a `ServerContext` or `CallbackServerContext` pointer or reference. For example, `ClientContext::FromServerContext` now uses a `ServerContextBase*` as its argument; this is not a breaking API change since the argument is now a parent class of the previous argument's class.

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

This has been implemented using the [core callback API](https://github.com/grpc/proposal/pull/181) as well as numerous pull requests in the [project associated with the callback API](https://github.com/grpc/grpc/projects/12).

## Open issues (if applicable)

N/A. The gRPC C++ callback API has been used internally at Google for nearly 2 years now and the code and API have evolved substantially during that period.

