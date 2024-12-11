L120: Remove core function gpr_atm_no_barrier_clamped_add
----
* Author(s): ctiller
* Approver: markdroth
* Status: In Review
* Implemented in: C++
* Last updated: 2024/12/10
* Discussion at: https://groups.google.com/g/grpc-io/c/YnxuoZdRH-Y

## Abstract

Remove gpr_atm_no_barrier_clamped_add.

## Background

There are no uses of this function remaining in gRPC Core, the function is buggy as implemented and we don't want to fix it.

## Proposal

Remove gpr_atm_no_barrier_clamped_add.

## Implementation

https://github.com/grpc/grpc/pull/38263

## Open issues (if applicable)
