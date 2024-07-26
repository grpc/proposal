Title
----
* Author(s): ctiller
* Approver: wenbozhu
* Status: Draft
* Implemented in: C++, Objective-C
* Last updated: [Date]
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Remove the Cronet transport from gRPC core (and consequently the Objective-C bindings to it).

## Background

gRPC C++, Objective-C can use Cronet on iOS as an alternative HTTP/2 stack.
The Cronet library itself has been deprecated since September 2023, and so instead of maintaining this code we're opting to remove it.

## Proposal

Remove the Cronet transport, and related APIs in core and Objective-C:
- `grpc_cronet_secure_channel_create`, and the containing header `grpc_cronet.h`
- `CronetChannelCredentials` in the C++ API
- `GRPCCall+Cronet.{h,mm}`, `GRPCCronetChannelFactory.{h,mm}` and all API therein
- `gGRPCCoreCronetID`

## Rationale

The Cronet team no longer maintains this code, so any bugs are going to be up to the gRPC team to fix, and gRPC does not have the expertise to maintain this stack.

## Implementation

A single PR will be submitted to remove the bulk of the transport. Follow-ups may be submitted to garbage collect missed pieces later.
