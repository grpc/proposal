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

* The API structure has substantial similarities to the gRPC-Node and gRPC-Java APIs

## Proposal

The most general form of the callback API is built around a _reactor_ model. Each type of RPC has a reactor base class provided by the library. These types are:

* `ClientUnaryReactor` for unary calls
* `ClientBidiReactor` for bidi-streaming 
* `ClientReadReactor` for server-streaming 
* `ClientWriteReactor` for client-streaming 
* `ServerUnaryReactor` for unary calls
* `ServerBidiReactor` for bidi-streaming 
* `ServerReadReactor` for client-streaming 
* `ServerWriteReactor` for server-streaming 

These base classes provide three types of methods:

* Operation-initiation methods: start an asynchronous activity in the RPC
   - StartRead, StartWrite, StartWritesDone: streaming operations as appropriate to the RPC variety
   - StartSendInitialMetadata: for server responses where the application wants to send the metadata separately from the first response message (uncommonly used)
   - Finish: for server RPCs to provide completion status. This initiates the asynchronous transmission of the status to the client.
* RPC completion prevention methods
   - AddHold, RemoveHold: for client RPCs, prevents the RPC from being considered complete until each AddHold is matched to a corresponding RemoveHold
* Operation-completion reaction methods: notification of completion of asynchronous RPC activity
   - OnReadDone, OnWriteDone, OnWritesDoneDone: methods invoked by the library at the completion of an explicitly-specified streaming RPC operation. Provides a `bool ok` argument to indicate whether the completion was successful.
   - OnReadInitialMetadataDone: notifying client RPCs that the server has sent initial metadata. It has a bool `ok` argument to indicate whether the completion was successful. This is automatically invoked by the library for all RPCs; it is uncommonly used however
   - OnCancel: for server RPCs only, called if an RPC is canceled before it has a chance to successfully send status to the client side
   - OnDone: called when all outstanding RPC operations are completed. Provides status on the client side.

As a shortcut, client-side unary RPCs _may_ bypass the reactor model by directly providing a `std::function` for the library to call at completion. This is semantically equivalent to a reactor in which the `OnDone` function simply invokes the specified function (but can be implemented in a slightly faster way since such an RPC will definitely not wait separately for initial metadata from the server). In practice, this is the common and recommended model for client-side unary RPCs, unless they have a specific need to wait for initial metadata before getting their full response message.

Server-side unary RPCs have the option of returning a library-provided default reactor when there method handler is invoked. This default reactor provides a Finish method, but does not provide a user callback for `OnCancel` and `OnDone`. In practice, this is the common and recommended model for most server-side unary RPCs unless they specifically need to react to an `OnCancel` callback or do cleanup work after the RPC fully completes.

## Rationale

## Implementation

## Open issues (if applicable)

N/A. The gRPC C++ callback API has been used internally at Google for nearly 2 years now and the code and API have evolved substantially during that period.

