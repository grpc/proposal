L113: C-core: Remove `grpc_channel_num_external_connectivity_watchers()`
----
* Author(s): @markdroth
* Approver: @ctiller
* Status: Implemented
* Implemented in: C-core
* Last updated: 2024-02-07
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will remove the `grpc_channel_num_external_connectivity_watchers()`
function from the C-core API.

## Background

This function is not really useful for wrapped languages.  Currently,
its only callers are in tests and do not appear to actually be
necessary.

In addition, we are in the process of making some implementation changes
that would be simplified by not having to support this function.

## Proposal

We will remove the `grpc_channel_num_external_connectivity_watchers()`
function from the C-core API.

## Implementation

Implemented in https://github.com/grpc/grpc/pull/35840.
