Static strings for `method` and `host` for the input to `grpc_channel_register_call()`
----
* Author(s): [Soheil Hassas Yeganeh](https://github.com/soheilhy)
* Approver: vjpai, yashykt
* Status: In Review
* Implemented in: core
* Last updated: 2019-06-05
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal proposes a requirement to only accept static strings for
`method` and `host` as the arguments to `grpc_channel_register_call()`.

## Background

For pre-registering, `method` and `host` are pre-registered on channels using
`grpc_channel_register_call()`. Although no user that we are aware of uses
non-static strings for `method` and `host`, we have to conservatively intern the
strings since `grpc_channel_register_call()` is not clear about the ownership
and life-time of `method` and `host`. Interning `method` and `host` results in
unnecessary ref-counting and cache-line contention for every single call. This
can be eliminated by making static.

## Proposal

Require static string for the `method` and `host` arguments to
`grpc_channel_register_call()`. That is the `method` and `host` strings must
be kept valid while the server is up.

## Rationale

`method` and `host` are reffed for every call. Doing an atomic fetch-add per
call is quite expensive, and truly unnecessary since the strings can be easily
kept alive while the channel is alive.

## Implementation

This is basically a simple comment update and removing `grpc_slice_intern()`
calls for `mehtod` and `host`, as implemented in
[PR #19263](https://github.com/grpc/grpc/pull/19263).
