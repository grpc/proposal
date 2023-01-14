L104: C-core: Ban GRPC_OP_SEND_STATUS_FROM_SERVER in combination with recv ops
----
* Author(s): ctiller
* Approver: markdroth
* Status: In Review
* Implemented in: C Core
* Last updated: 2022/11/03
* Discussion at: https://groups.google.com/g/grpc-io/c/MAxqC_LI5O8

## Abstract

Ban the combination of ops `GRPC_OP_SEND_STATUS_FROM_SERVER` and any of `GRPC_OP_RECV_MESSAGE`, `GRPC_OP_RECV_CLOSE_ON_SERVER`.


## Background

Sending status from a server is a request for a full close of that call - neither reads not writes can proceed after it occurs.
As such, it's unclear what behavior should occur when these operations are combined.
No official API allows the creation of batches with these combinations - but several low level tests (below the canonical bindings) leverage the ability to combine these operations, presumably for brevity of test implementation.

With the conversion to promise based APIs internally, with their formulation that the promise completes with send status and then no longer does anything, is at odds with the implementation quirks that are load bearing in this minority of tests.
Two solutions present themselves: this proposal, or transparently breaking the batch in two (presumably in call.cc) and sending the non `GRPC_OP_SEND_STATUS_FROM_SERVER` ops first, followed by a second batch with just `GRPC_OP_SEND_STATUS_FROM_SERVER`, and transparently merging the completions before reporting them up.
Though the latter change is more self contained, the former matches better with gRPC's overall semantics and can be performed as a one time test refactoring.


## Proposal

If a batch passed to grpc_call_start_batch contains:
- `GRPC_OP_SEND_STATUS_FROM_SERVER`
- and `GRPC_OP_RECV_MESSAGE` _or_ `GRPC_OP_RECV_CLOSE_ON_SERVER`
instead of processing it, return `GRPC_CALL_ERROR`.


## Implementation

https://github.com/grpc/grpc/pull/31554 includes a change that tests for the objectionable condition. With the test in place we'll iterate on updating tests until all tests pass with this new restriction.
