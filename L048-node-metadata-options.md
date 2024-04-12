Node gRPC Metadata options
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node
* Last updated: 2019-03-01
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add options to the Metadata class corresponding to start-of-call flags in the core.

## Background

The gRPC core library defines a few "metadata" flags that can be passed to `grpc_call_start_batch` at the beginning of a call in a "send initial metadata" `grpc_op`. Currently, there is no way for a user of the Node gRPC library to set those flags.

## Proposal

Currently the `Metadata` constructor takes no arguments. We will modify it to take one argument which is an "options" object mapping string keys to boolean values. These options correspond to the core's [initial metadata flags](https://github.com/grpc/grpc/blob/a4b8667de96f8be14236a7312dca52c492a6d159/include/grpc/impl/codegen/grpc_types.h#L432) The following options will be accepted:

 - `idempotentRequest`: Signal that the call is idempotent. Default: `false`
 - `waitForReady`: Signal that the call should not return UNAVAILABLE before it has started. Default: `true`
 - `cacheableRequest`: Signal that the call is cacheable. GRPC is free to use GET verb. Default: `false`
 - `corked`: Signal that the initial metadata should be corked. Default: `false`

We will also add a `setOptions` method to the `Metadata` class that accepts the same options object and replaces the previously set options.

## Rationale

These flags need to be passed to the core in `grpc_call_start_batch` in a "send initial metadata" `grpc_op`, so they can be passed if and only if metadata is also passed. The simplest way to accomplish this is to make these flags part of the `Metadata` structure. It is useful to include a setter to allow users to easily use the same metadata with different calls that would want to use different flags.


## Implementation

I (murgatroid99) will implement this in both `grpc` and `@grpc/grpc-js`. The initial implementation in `@grpc/grpc-js` will have no behavioral changes from the options.
