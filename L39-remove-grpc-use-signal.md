Remove grpc_use_signal from core surface API
----
* Author(s): vjpai
* Approver: AspirinSJL
* Status: Proposed
* Implemented in: https://github.com/grpc/grpc/pull/16706
* Last updated: 2018-09-26
* Discussion at: 

## Abstract

Remove `grpc_use_signal` from the core surface API

## Background

The `epollsig` polling engine used signals to kick polling
threads. This polling engine has been deprecated for a while and has
been [deleted](https://github.com/grpc/grpc/pull/16679). The
`grpc_use_signal` surface API allowed code to prevent the use of
signals or to change the signal number used by `epollsig`. With the
deletion of this polling engine, gRPC core no longer uses signals of
any kind, and this API should be deleted.


### Related Proposals:

N/A

## Proposal

Delete the surface API and bump the core version number.

## Rationale

Not only is this surface API no longer useful, I cannot find any
example of it being used.

## Implementation

Core: https://github.com/grpc/grpc/pull/16706

## Open issues (if applicable)

N/A
