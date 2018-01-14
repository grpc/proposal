Delete grpc_alarm from gRPC core API
----
* Author(s): vjpai
* Approver: a11r
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/pull/14015
* Last updated: January 14, 2018
* Discussion at:

## Abstract

Remove `struct grpc_alarm` and `grpc_alarm_*` functions from the core API

## Background

`grpc::Alarm` [was introduced as a gRPC C++ API](https://github.com/grpc/grpc/pull/3618) to inject completion queue events at specified times because it was observed to be [difficult to manage timing-based code in the asynchronous API](https://github.com/grpc/grpc/pull/1949). This was implemented by creating a
matching `grpc_alarm` in core which internally used the existing `grpc_timer` as its
implementation; the `grpc::Alarm` in C++ was only a thin wrapping around `grpc_alarm`.

### Related Proposals:

This is related to the overall project of [de-wrapping C++](https://github.com/grpc/grpc/projects/8).

## Proposal

* Remove `grpc_alarm` and related functions from gRPC Core.
* Re-implement `grpc::Alarm` by directly invoking gRPC Core sub-surface features such as `grpc_timer`

## Rationale

`grpc::Alarm` has been used by external projects. However, `grpc_alarm` has not been used even by any wrapped language beside C++.  Thus, it is not needed in core, and removing it from core allows additional flexibility in its C++ implementation.

## Implementation

https://github.com/grpc/grpc/pull/14015 implements this change.

## Open issues (if applicable)

N/A
