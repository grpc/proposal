L105: Python Add New Error Types
----
* Author(s): XuanWang-Amos
* Approver: gnossen
* Status: In Review
* Implemented in: Python
* Last updated: 08/21/2023
* Discussion at: TODO

## Abstract

Add two new errors in grpc public API:
  * BaseError
  * AbortError

## Background

In case of abort, currently we don't log anything, exposing those error types allows user to catch and handle aborts if they want.

## Proposal

Add AbortError and BaseError in public API.

## Rationale

The Async API [has similar errors](https://github.com/grpc/grpc/blob/v1.57.x/src/python/grpcio/grpc/aio/__init__.py#L23,L24). We're refactoring code so those errors will also be used in Sync API. Adding them to the Sync API will help us keep the two stacks in sync and allow users of the Sync implementation to catch and handle aborts.

## Implementation

And and check AbortError while abort : https://github.com/grpc/grpc/pull/33969
