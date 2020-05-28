Core server RPCs will not report cancellation if completed with non-OK status
----
* Author(s): vpai
* Approver: markdroth
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/pull/22991
* Last updated: May 28, 2020
* Discussion at https://groups.google.com/g/grpc-io/c/5o3EDPqz9is

## Abstract

Clarify that gRPC core will not mark server RPCs cancelled if they explicitly close with `GRPC_OP_SEND_STATUS_FROM_SERVER` using a non-`OK` status.  This is technically a core API change and thus needs a major revision increment.

## Background

gRPC core versions up to 10.X specify that the out argument of the `GRPC_OP_RECV_CLOSE_ON_SERVER` operation is a pointer to an int that specifies whether the RPC was cancelled. The definition specified in the comment is

```
      /** out argument, set to 1 if the call failed in any way (seen as a
          cancellation on the server), or 0 if the call succeeded */
```

The term `in any way` is not clear, and the broadest (and current) definition is that it accounts for failures caused by client-side or server-side cancellations, deadline exceeded, maximum connection age exceeded, network resets, *and also explicit sending of non-OK status*.


### Related Proposals

N/A

## Proposal

Clarify that RPCs will only be marked cancelled if they failed for a reason other than completion with an explicit non-OK status provided using the `GRPC_OP_SEND_STATUS_FROM_SERVER` operation. In practice, this means that RPCs to be considered cancelled are those for which the server was not able to successfully send any kind of status to the client (because the RPC was explicitly cancelled, the deadline was exceeded, because some HTTP configuration parameter like maximum connection age took effect, etc.).

## Rationale

Calling an RPC cancelled if it completes for entirely expected reasons is confusing. Generally speaking, this distinction has not been noticed at the API layers but will become more common once the C++ callback API becomes formalized (since that has an OnCancel reaction for server RPCs). As a result, this issue should be clarified and fixed before the C++ callback API becomes common.

## Implementation

1. Decide if a server call is canceled by whether or not it could successfully send status
1. Fix core tests (wrapped language tests already had the proper expectations)
1. Mark a call failed according to channelz if it was canceled or if the server sent a non-OK status
1. Add a C++ test to validate that OnCancel is not called on non-OK explicit status
 1. Fix the core surface comment about this topic and bump the core major API version

## Open issues (if applicable)

* Does this affect the core wrapped language APIs?
  - This distinction is not visible to Ruby or the C++ synchronous API at all since the RPC in both of those cases is complete when returning status. Thus cancellation checks just see the effect of other forms of cancellation.
  - For the C++ asynchronous API, `IsCancelled` currently doesn't define its result. This change will also include an explciit definition of cancellation for C++.
  - For Python, ???
  - For C#, ???

* Should this only trigger on explicit cancellations?
- In practice, it doesn't matter if the application requested cancellation or cancellation happened because of a deadline or network problem; in any case, the RPC is failed for reasons out of the scope of the RPC. In practice, issues like deadline exceeded could be implemented using cancellation, so there is no strong distinction in any case.

* Should server-side cancellations mark an RPC cancelled since they are an explicit part of the server operation?
- In practice, server-side completion is preferred over cancellation even on failed RPCs. Cancellation should be used for unusual or unexpected cases (such as a need to promptly release resources), so that should still mark RPCs cancelled. Additionally, the name would make it non-intuitive to treat server-side cancellations as anything other than cancelled RPCs.


