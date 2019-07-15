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
types conflict with types provided by one of our dependencies and which are used by our code.
In order to keep gRPC C# working with future .NET releases, we are forced to remove
the conflicting types from our codebase. See https://github.com/grpc/grpc/issues/18592
for more technical details.

## Proposal

This proposal has two parts, which should be considered as a whole. Change 1 (which is forced by the language ecosystem)
will necessarily lead to binary breaking change (users will need to recompile their code) so if Change 2
is made at the same time, there will be no extra cost to users, but there will be some significant benefits.
Therefore we will make both changes at the same time and release the next gRPC C# version as v2.23.0 (instead of v1.23.0).

We chose version v2.23.0 instead of v2.0.0 so that the minor version number can still be used for comparing how old a given release is relative to all other gRPC implementations. E.g. gRPC C# 2.24.x will be released together with gRPC C++ 1.24.x.

NO protocol changes are proposed between gRPC C# version 2.x and 1.x - both versions will be fully interoperable with each other and also with all other gRPC implementations.

#### Change 1: Remove the type that conflicts with .NET base class library

Remove the references to `System.Collections.Generic.IAsyncEnumerator<T>` (the type that's now in conflict
with .NET base class libraries) from our codebase. Methods declared by that interface will be moved to 
to an existing type `Grpc.Core.IAsyncStreamReader<T>` (which inherits from `IAsyncEnumerator<T>`).
This has the following consequences:
- any user code that doesn't explicitly reference `IAsyncEnumerator<T>` will continue working with
  no changes required, users will only need to recompile. We expect this is going to be the case for majority of users,
  as there is no real reason for gRPC users to use `IAsyncEnumerator<T>` directly.
- the user code that references `IAsyncEnumerator<T>` explicitly will need to change all the occurences to use `Grpc.Core.IAsyncStreamReader<T>` 
  instead. Note that offering a replacement type is the best we can do here as there is simply no way to avoid a source-level breakage in codebase that directly uses a type which needs to be removed.

Proposed PR: https://github.com/grpc/grpc/pull/19059

#### Change 2: Introduce common concept of a channel "ChannelBase" that can be used by different implementations

Motivation: gRPC currently has 2 implementations for .NET: [gRPC C#](https://github.com/grpc/grpc/tree/master/src/csharp) and [grpc-dotnet](https://github.com/grpc/grpc-dotnet). While based on 2 different network stack,
it is highly desirable that these two implementation share the same API for invoking (=client) and handling (=server) calls.
On the client side, the challenge is that the `Channel` API has been historically designed to accomodate some specifics
of the C-core based implementation, and as such it is difficult to use for the grpc-dotnet implementation in its current form. Therefore, the grpc-dotnet implementation currently doesn't expose the concept of a "Channel" explicitly. While this doesn't limit the basic functionality of grpc-dotnet clients, it will likely become a problem as more advanced features are added in the future. Therefore, we propose introducing a common concept of a channel that can be used by both implementations.

Proposed changes:
- Introduce a new `ChannelBase` class and make the current `Channel` class inherit from it. `ChannelBase` will represent the shared concept of a client-side channel.
- `ClientBase` (parent class for all generated client classes) will take `ChannelBase` in constructor instead of `Channel`.
  This qualifies as a binary-level breaking change (requires recompile).
  The advantages of making this breaking change is that it allows unifying the client channel side API for all gRPC implementations.
  
Proposed PR: https://github.com/grpc/grpc/pull/19599

## Rationale

#### Change 1

This is a change forced by the .NET ecosystem. Risk: If no action is taken, gRPC C# will stop working for .NET Core 3 users
and everyone else who wants/needs to use the newer version of BCL library. .NET Core 3 is a requirement for the grpc-dotnet implementation and it is expected that a majority of users will eventually migrate to .NET Core 3. Making a breaking change in this situation seems adequate and the objective is to make the users' transition as simple as possible (the proposed solution reflects that objective).

#### Change 2

Pros
- Provides a concept of channel that is implementation-independent and makes both .NET implementations of gRPC more aligned both conceptually and in terms of API. That seems to be a good investment for the future.
- `ClientBase` can be moved to the shared API package Grpc.Core.Api and be used by all gRPC implementations, which gives us a way to get rid of `LiteClientBase` which was introduced to provide an alternative solution, but brings some extra complexity 
- Auxilliary gRPC packages like Grpc.Reflection and Grpc.HealthCheck could previously be used only by one of the implementations. After the change, we can modify them to be usable with both gRPC C# and grpc-dotnet

Cons
- It's a binary-level breaking change, but we're forced to make a breaking binary change due to Change 1 anyway, so there's not extra cost to the user.

## Implementation

Most of proposed changes already exist in form of a PR
- `IAsyncEnumerator` dependency will be removed in https://github.com/grpc/grpc/pull/19059
- `ChannelBase` will be introduced in https://github.com/grpc/grpc/pull/19599
- jtattermusch will update the gRPC C# version for the next release to 1 -> 2

## Migration instructions for users

To upgrade from gRPC C# v1.x to v2.x:
- just upgrade the nuget packages and rebuild your project. Note that assemblies built against gRPC v1.x might not work against v2.x without rebuilding first.
- if your code contains any direct references to `IAsyncEnumerator<T>`, change all of them to `Grpc.Core.IAsyncStreamReader<T>` (which has exactly the same members as the original interface) and rebuild your project.

The migration is fully transparent to the remote peers - gRPC C# version 2.x and 1.x are fully interoperable with each other and also with all other gRPC implementations.
