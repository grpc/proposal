Node gRPC Types Package
----
* Author(s): mlumish
* Approver: wenboz
* Status: Draft
* Implemented in: Node.js
* Last updated: 2019-02-28
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/30SrKiqsEis

## Abstract

This proposal defines a Node package that defines the TypeScript types and some simple JavaScript objects that can be used to define a package that is intended to be able to plug in to both `grpc` and `@grpc/grpc-js`.

## Background

Some packages are intended to plug in to grpc without directly making grpc requests, and without depending on the particular implementation. This includes packages that provide service definitions and/or implementations, such as `grpc-health-check`, and packages that make behavioral modifications such as `grpc-gcp`. These packages need access to TypeScript type definitions for gRPC and to certain constants such as status codes. Currently, that information is only exported by the gRPC implementation packages, so packages like that currently needs to depend on a gRPC implementation package to work.


### Related Proposals: 
* A list of proposals this proposal builds on or supersedes.

## Proposal

We will create a new package `@grpc/types` that exports TypeScript type definitions and some JavaScript data. The TypeScript type definitions will be derived from the `@grpc/grpc-js` type definitions as follows:

 - `type`, `interface`, and `enum` definitions will be reproduced verbatim
 - `interface` definitions will be created corresponding to classes exported by `@grpc/grpc-js`
 - An interface named `grpc` will be exported with members defined as follows:
   - Function `type` definitions corresponding to functions exported by `@grpc/grpc-js`
   - constructor `interface` definitions corresponding to classes exported by `@grpc/grpc-js`

 This will be compiled to create a JavaScript file that only exports the `enum` types.

 After `@grpc/types` is published `grpc` and `@grpc/grpc-js` will be modified to depend on it and define their types in terms of its types to ensure consistency.

 The intention is that plugin libraries that require access to concrete functions or classes from the gRPC API would request that their users pass a reference to that function or class from the gRPC implementation library that user is using the plugin with. For the benefit of TypeScript type matching, plugins that require access to classes would probably need to request factory functions instead.

## Rationale

In order to make the TypeScript types for gRPC available to plugin libraries independent of the implementation, they need to be distributed in a separate package. We would only include `interface`, `type`, and `enum` definitions in this package because the goal is to describe an abstract interface that matches the individual type definitions in the implementation libraries.

Defining a `grpc` interface with all of the different members allows plugin libraries to have a proper type to use if they want to ask users for the entire implementation library exports object. In other words it would allow plugin libraries to properly typecheck an API that asks users to do something like the following:

```ts
import * as grpc from 'grpc' // or '@grpc/grpc-js'
import * as pluginLibrary from 'plugin-library'
pluginLibrary.initializeWith(grpc);
```


## Implementation

I (murgatroid99) will implement and publish this library, then I will make the necessary modifications to the `grpc` and `@grpc/grpc-js` library to use these types.
