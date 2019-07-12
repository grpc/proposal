gRPC C# 2.x Release 
----
* Author(s): jtattermusch
* Approver: a11r
* Status: Draft
* Implemented in: C#
* Last updated: 2019-07-11
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC C# implementation will do a breaking change in the next release, which
also forces a major version increase because we follow semantic versioning.
This document summarizes what are the changes being made and the reasons why they are needed.
It also describes the impact on the users and documents the migration steps.

## Background

.NET Core framework is adding new types in the next release, but these
types conflict with types that are used by one of our dependencies and by our code.
In order to keep gRPC C# working with future .NET releases, we are forced to remove
the conflicting types from our codebase. See https://github.com/grpc/grpc/issues/18592
for more technical details.

## Proposal

This proposal has two parts, which should be considered as a whole. Change 1 (which is forced by the language ecosystem)
will necessarily lead to binary breaking change (users will need to recompile their code) so if Change 2
is made at the same time, there will be no extra cost to users, but there will be some significant benefits.
Therefore we do both changes at the same time and release the next gRPC C# version as  
v2.23.0 (instead of v1.23.0).

#### Change 1: Remove the type that conflicts with .NET base class library

Remove the references to `System.Collections.Generic.IAsyncEnumerator<T>` (the type that's now in conflict
with .NET base class libraries) from our codebase. Methods declared by that interface will be moved to 
to an existing type `Grpc.Core.IAsyncStreamReader<T>` (which inherits from `IAsyncEnumerator<T>`).
This has the following consequences
- any user code that doesn't explicitly explicitly reference `IAsyncEnumerator<T>` will continue working with
  no changes required, users will only need to recompile. We expect this is going to be the case for majority of users,
  as there is no real reason for gRPC users to use IAsyncEnumerator<T> directly.
- the user code that references `IAsyncEnumerator<T>` explicitly will need to change all the occurences to use `Grpc.Core.IAsyncStreamReader<T>` 
  instead. Note that this is the best we can do as there is no way to prevent avoid a breakage if a type gets removed.
 
The exact way of making the change is in https://github.com/grpc/grpc/pull/19059

#### Change 2: Introduce common concept of a channel that can be used by different implementations

gRPC currently has 2 implementations: gRPC C# and grpc-dotnet. While based on 2 different network stack,
it is highly desirable that these two implementation share the same API for invoking (=client) and handling (=server) calls.
On the client side, the challenge is that the `Channel` API has been historically designed to accomodate some specifics
of the C-core based implementation, and as such it is difficult to use for the grpc-dotnet implementation in its current form.
Therefore, the grpc-dotnet implementation currently doesn't expose the concept of a "Channel" explicitly. This doesn't limit
the basic functionality of grpc-dotnet clients, but it will likely become a problem as more advanced features are being added
in the future. Therefore, we propose introducing a common concept of a channel that can be used by both implementations.

Proposed changes:
- introduce a new `ChannelBase` class and make the current `Channel` class inherit from it
- 
https://github.com/grpc/grpc/pull/19599/files the breaking change: ClientBase constructor now takes a ChannelBase


from our codebase by moving

advantages:
ClientBase can be moved to Grpc.Core.Api (not done yet in this PR)
LiteClientBase can be removed (not done yet in this PR)   (need to change constructor in generated code).
Client packages like Grpc.Reflection can now depend Grpc.Core.Api. They will work with C-Core client and .NET HttpClient client. (but code needs to be regenerated)
only a binary breaking change (= recompiling the code will fix things).


## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

