L105: Python Add New Error Types
----
* Author(s): XuanWang-Amos
* Approver: gnossen
* Status: In Review
* Implemented in: Python
* Last updated: 08/31/2023
* Discussion at: https://groups.google.com/g/grpc-io/c/pG34X9nAa3c

## Abstract

* Add two new errors in grpc public API:
  * BaseError
  * AbortError
* Also changed RpcError to be a subclass of BaseError.

## Background

In case of abort, currently we don't log anything, exposing those error types allows user to catch and handle aborts if they want.

## Proposal

1. Add AbortError and BaseError in public API.
2. Change RpcError to be a subclass of BaseError.


## Rationale

The Async API [has similar errors](https://github.com/grpc/grpc/blob/v1.57.x/src/python/grpcio/grpc/aio/__init__.py#L23,L24). We're refactoring code so those errors will also be used in Sync API. Adding them to the Sync API will help us keep the two stacks in sync and allow users of the Sync implementation to catch and handle aborts.

We also plan to change RpcError to be a subclass of BaseError so that all grpc errors are a subclass of BaseError, this will allow users to catching all gRPC exceptions use code like this:

```Python
try:
  do_grpc_stuff()
except grpc.BaseError as e:
  # handle error
```

## Implementation

And and check AbortError while abort : https://github.com/grpc/grpc/pull/33969
