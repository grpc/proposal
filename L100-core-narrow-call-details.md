L100: C-Core Narrow grpc_call_details
----
* Author(s): ctiller
* Approver: markdroth
* Status: In Review
* Implemented in: C Core
* Last updated: [Date]
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Remove unused fields from grpc_call_details.

## Background

grpc_call_details contains two fields that are both always zero.
Remove them and the need to test that they are so.


### Related Proposals: 

None.

## Proposal

From grpc_call_details:
- Remove the field `flags`.
- Remove the field `reserved`.

## Rationale

These fields must always be set to zero and provide no information.

## Implementation

Implemented as part of https://github.com/grpc/grpc/pull/30444.
