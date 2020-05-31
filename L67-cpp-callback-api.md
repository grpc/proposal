C++ callback-based asynchronous API
----
* Author(s): vjpai, sheenaqotj
* Approver: 
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/projects/12
* Last updated: May 28, 2020
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

The goal of the synchronous version is to be easy to program. However, this comes at the cost of high thread-switching overhead and high thread storage for systems with many concurrent RPCs. On the other hand, the asynchronous API allows the application full control over its threading and thus can scale further. It does, however, present challenges in deciding how many completion queues to use and how many threads to use for polling them, as one can either optimize for reducing thread hops, avoiding stranding, reducing CQ contention, or improving locality. These goals are often in conflict and require substantial tuning.

### Related proposals

* The [EventManager](https://www.github.com/grpc/proposal/XXX) is critical to the operation of the callback API.
* The API structure has substantial similarities to the gRPC-Node and gRPC-Java APIs

## Proposal

The callback API is designed to have the performance and thread scalability of an asynchronous API without the burdensome programming model of the completion-queue-based model. In particular, the following are fundamental guiding principles of the API:

* Library directly calls user-specified code at the completion of RPC actions.
* No explicit polling required for notification of completion of RPC actions.
   - In practice, these requirements mean that there must be a library-controlled poller for monitoring such actions. This _EventManager_ is described in another gRFC.
* As in the synchronous API, server RPCs have an application-defined method handler function as part of their service definition. The library invokes this method handler when a new server RPC starts.
* LIke the synchronnous API and unlike the completion-queue-based asynchronous API, there is no need for the application to "request" new server RPCs. Server RPC context structures will be allocated and have their resources allocated as and when RPCs arrive at the server.

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
   - `StartRead`, `StartWrite`, `StartWritesDone`: streaming operations as appropriate to the RPC variety
   - `StartSendInitialMetadata`: for server responses where the application wants to send the metadata separately from the first response message (uncommonly used)
   - `Finish`: for server RPCs to provide completion status. This initiates the asynchronous transmission of the status to the client.
1. Operation-completion reaction methods: notification of completion of asynchronous RPC activity. These are all virtual methods that default to an empty function (i.e., `{}`) but may be overridden by the application's reactor definition.
   - `OnReadDone`, `OnWriteDone`, `OnWritesDoneDone`: methods invoked by the library at the completion of an explicitly-specified streaming RPC operation. Provides a `bool ok` argument to indicate whether the completion was successful.
   - `OnReadInitialMetadataDone`: notifying client RPCs that the server has sent initial metadata. It has a bool `ok` argument to indicate whether the completion was successful. This is automatically invoked by the library for all RPCs; it is uncommonly used however
   - `OnCancel`: for server RPCs only, called if an RPC is canceled before it has a chance to successfully send status to the client side
   - `OnDone`: called when all outstanding RPC operations are completed. Provides status on the client side. 
1. RPC completion prevention methods. These are methods provided by the class and are not virtual.
   - `AddHold`, `RemoveHold`: for client RPCs, prevents the RPC from being considered complete (ready for `OnDone`) until each AddHold is matched to a corresponding RemoveHold. These are used to perform _extra-reaction flows_, which refers to  streaming operations initiated from outside a reaction method or method handler.

### Unary RPC shortcuts

As a shortcut, client-side unary RPCs _may_ bypass the reactor model by directly providing a `std::function` for the library to call at completion rather than a reactor object pointer. This is semantically equivalent to a reactor in which the `OnDone` function simply invokes the specified function (but can be implemented in a slightly faster way since such an RPC will definitely not wait separately for initial metadata from the server). In practice, this is the common and recommended model for client-side unary RPCs, unless they have a specific need to wait for initial metadata before getting their full response message.

Server-side unary RPCs have the option of returning a library-provided default reactor when there method handler is invoked. This default reactor provides a Finish method, but does not provide a user callback for `OnCancel` and `OnDone`. In practice, this is the common and recommended model for most server-side unary RPCs unless they specifically need to react to an `OnCancel` callback or do cleanup work after the RPC fully completes.

### ServerContext extensions

`ServerContext` is now made a derived class of `ServerContextBase`. There is a new derived class of `ServerContextBase` called `CallbackServerContext` which provides a few additional functions:

* `DefaultReactor` may be used by a method handler to return a default reactor from a unary RPC.
* `GetRpcAllocatorState`: see advanced topics section

Additionally, the `AsyncNotifyWhenDone` function is not present in the `CallbackServerContext`. All method handler functions for the callback API take a `CallbackServerContext*` as their first argument. `ServerContext` (used for the sync and CQ-based async APIs) and `CallbackServerContext` (used for the callback API) actually use the same underlying structure and thus their object pointers are meaningfully convertible to each other via a `static_cast` to `ServerContextBase*`. We recommend that any helper functions that need to work across API variants should use a `ServerContextBase` pointer or reference as their argument rather than a `ServerContext` or `CallbackServerContext` pointer or reference.

### Advanced topics

Message allocator

`RegisterCallbackGenericService` is a new method of `ServerBuilder` to allow for processing of generic (unparsed) RPCs. This is similar to the pre-existing `RegisterAsyncGenericService` but uses the callback API and reactors rather than the CQ-based async API. It is expected to be used primarily for generic gRPC proxies where the exact serialization format or list of supported methods is unknown.

Method-by-method specification of APi type

## Rationale

## Implementation

## Open issues (if applicable)

N/A. The gRPC C++ callback API has been used internally at Google for nearly 2 years now and the code and API have evolved substantially during that period.

