Title
----
* Author(s): ctiller@google.com
* Approver: roth@google.com
* Status: Draft
* Implemented in: Core
* Last updated: 1/7/2022
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Remove slice interning related APIs.

## Background

gRPC Core used to rely on slice interning to improve metadata handling performance.
Recent advancements in the metadata system design have obviated the need for this, and so it's time to remove the API complexity incurred by this feature.

### Related Proposals: 
None.

## Proposal

Remove the following APIs:
* grpc_slice_intern
* grpc_slice_default_hash_impl
* grpc_slice_default_eq_impl
* grpc_slice_hash

The first removes the ability to create an interned slice.
Because interned slices had opinions about hashing and equality (since they could optimize those operations) we introduced hashing and equality hooks when interning was introduced.
These hooks are no longer required, so we'll remove the gRPC opinion on how slices should be hashed - wrapped languages are of course free to choose their own hash function for slice data.

## Rationale

We don't need this anymore.

## Implementation

This is implemented in https://github.com/grpc/grpc/pull/28363.
