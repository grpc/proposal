L84: Add grpc_call_failed_before_recv_message() to C-core API
----
* Author: alishananda
* Approver: ctiller
* Status: In Review
* Implemented in: C++
* Last updated: 2021-08-27
* Discussion at: https://groups.google.com/g/grpc-io/c/UNBHGr6feeY

## Abstract

There is a race between `ServerContext::IsCancelled` and `Read` in the sync API where a call is cancelled and `Read` returns false but `IsCancelled` also returns false. We need to ensure we propagate the cancellation to `IsCancelled` so it always returns true if `Read` returns false and the call was cancelled.

As part of this fix, we have to move `grpc_call_failed_before_recv_message` into the C-core surface API so the implementation of `MaybeMarkCancelledOnRead` can be inlined and found by proto_library.

## Background

A user reported that after cancelling a call, `IsCancelled` would sometimes return false when `Read` would also return false. 

We saw this same issue with the callback API, where the server couldn't tell whether `OnReadDone` returned false because the stream was cancelled or actually closed cleanly. [This PR](https://github.com/grpc/grpc/pull/26245) added a core subsurface `grpc_call_failed_before_recv_message` function in `src/core/lib/surface/call.h` and `MaybeMarkCancelledOnRead` function depending on this value in `src/cpp/server/server_context.cc`. Together, these distinguish a cancellation from a cleanly closed stream by marking places in the transport where `OnReadDone` will return an empty message because of a stream failure and using `grpc_call_failed_before_recv_message` to propagate this information out of the grpc_call struct.

### Related Proposals: 
N/A

## Proposal

The proposal is to move `grpc_call_failed_before_recv_message` from subsurface to the C-core surface API.

## Rationale

It seems that the build structure of the callback API is different than that of the sync API. When using `MaybeMarkCancelledOnRead` with the Sync API, the proto library returns a linker error that it cannot find the function because the proto library does not depend on the file implementing the function. We thus have to inline `MaybeMarkCancelledOnRead` in `server_context.h`, which will mean adding the `grpc_call_failed_before_recv_message` function to our public API.


## Implementation

We will move `grpc_call_failed_before_recv_message` out of `call.h` and into the C-core surface API.

The PR with this implementation is https://github.com/grpc/grpc/pull/27056.
