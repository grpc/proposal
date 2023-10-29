Node.js Reflection Server Library
----
* Author(s): jtimmons
* Approver: murgatroid99
* Status: In Review
* Implemented in: Node.js
* Last updated: 2023-10-11
* Discussion at: https://groups.google.com/g/grpc-io/c/Ie2jFIHCwrc

## Abstract

Create a canonical implementation of the gRPC reflection API for Node.js based on the logic from the [nestjs-grpc-reflection library](https://gitlab.com/jtimmons/nestjs-grpc-reflection-module/-/blob/30b67a78ff99e31ae54a0ab34c3784316579c665/src/grpc-reflection/grpc-reflection.service.ts)

## Background

Since its introduction in 2017 there have been a variety of external node.js implementations for the [gRPC Reflection Specification](https://github.com/grpc/grpc/blob/ce75ec23a1a9c5239834b92da4ce0992d367a39c/doc/server-reflection.md), each of which is in various states of maintenance. A few examples are linked at the bottom of this section.

This feature was initially requested in grpc/grpc-node#79

* https://gitlab.com/jtimmons/nestjs-grpc-reflection-module
* https://github.com/deeplay-io/nice-grpc/tree/master/packages/nice-grpc-server-reflection
* https://github.com/AckeeCZ/grpc-server-reflection

### Related Proposals:
* Initial reflection proposal: https://github.com/grpc/proposal/blob/master/A15-promote-reflection.md
* Proposal of API for similar library: https://github.com/grpc/proposal/blob/master/L106-node-heath-check-library.md

## Proposal

We are proposing the creation of a new `@grpc/reflection` package with the following external interface:

```ts
import type { Server as GrpcServer } from '@grpc/grpc-js';
import type { PackageDefinition } from '@grpc/proto-loader';

type MinimalGrpcServer = Pick<GrpcServer, 'addService'>;

interface ReflectionServerOptions {
  services?: string[]; // whitelist of fully-qualified service names to expose. Default: expose all
}

export interface ReflectionServer {
  constructor(pkg: PackageDefinition, options?: ReflectionServerOptions);
  addToServer(server: MinimalGrpcServer);
}
```

this `ReflectionServer` class will be used to expose information about the user's gRPC package according to the gRPC Reflection Specification via their existing gRPC Server. Internally, the class will be responsible for managing incoming requests for each of the various published versions of the gRPC Reflection Specification: at the time of writing, this includes `v1` and `v1alpha` but may include more in the future. These version-specific handlers will be isolated into their own services in order to preserve backwards-compatibility, and will look like the following:

**reflection.v1.ts**
```ts
import {
  ExtensionNumberResponse,
  FileDescriptorResponse,
  ListServiceResponse,
} from './proto/grpc/reflection/v1/reflection';

export interface ReflectionV1Implementation {
  constructor(pkg: PackageDefinition);

  listServices(listServices: string): ListServiceResponse;
  fileContainingSymbol(symbol: string): FileDescriptorResponse;
  fileByFilename(filename: string): FileDescriptorResponse;
  fileContainingExtension(symbol: string, field: number): FileDescriptorResponse;
  allExtensionNumbersOfType(symbol: string): ExtensionNumberResponse;
}
```

### Usage
The user will leverage the library in a way very similar to the [gRPC health check service](https://github.com/grpc/grpc-node/tree/83743646cf69baf9ae1294015de5ffed33339154/packages/grpc-health-check) by creating a new class to manage the reflection logic and then adding that to the gRPC server:

```ts
import { join } from 'path';

import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import { ReflectionServer } from '@grpc/reflection';

const pkg = protoLoader.loadSync(join(__dirname, 'sample.proto'));
const reflection = new ReflectionServer(pkg);

const server = new grpc.Server();
const proto = grpc.loadPackageDefinition(pkg) as any;
server.addService(proto.sample.SampleService.service, { ... });
reflection.addToServer(server)

server.bindAsync('0.0.0.0:5001', grpc.ServerCredentials.createInsecure(), () => { server.start(); });
```

## Rationale

### 1. Design Decision: use of proto-loader over protoc
several reflection implementations linked above leverage protoc in order to generate a representation of the proto schema to expose on the API. In this document we propose the use of proto-loader to inspect the schema at runtime instead in order to simplify the developer experience and be consistent with the [design of the `grpc-health-check` library](https://github.com/grpc/proposal/blob/ee75a4010214ddda02ba992e69f1c57be7f71497/L106-node-heath-check-library.md#switch-from-protoc-to-grpcproto-loader).

### 2. Design Decision: support multiple reflection implementations
currently not all reflection clients request the `v1` version of the spec so we need to include handlers for both `v1` and `v1alpha` to support both during the transition. For this reason we separate the reflection handling logic itself to allow for reuse across multiple service versions

### 3. Design Decision: restrict user to loading a single PackageDefinition

ideally we would be able to support the user adding multiple `PackageDefinition` objects at a time in a similar way to the gRPC server itself, however due to some internal protobuf behavior discussed in [this thread](https://github.com/grpc/proposal/pull/397#discussion_r1357181337) this is currently difficult to accomplish. For that reason we will be restricting the user to loading a single PackageDefinition for the time being to avoid any confusing behavior or bugs. Detail on the issue is described below for completeness:

**background**: when loading a `PackageDefinition` object via the `protoLoader.load(...)` function, proto-loader/protobufjs will rename the input `.proto` files based on their [protobuf package](https://protobuf.dev/programming-guides/proto3/#packages) name. For example a file named _file.proto_ in the `sample` package will actually be referred to as _sample.proto_ in all `FileDescriptorProto` objects in the resulting `PackageDefinition`.

This behavior can cause issues when multiple files exist within the _same_ package as there can be confusion about what is the "real" contents of a file (which is critical information for the reflection API). Proto-loader/protobufjs handles this for a _single_ `load()` call by unifying all files into a single package-file; for example: if we have files _vendor/a.proto_ and _vendor/b.proto_ which are both in the `vendor` protobuf namespace then contents from both files will be combined into a single _vendor.proto_ file descriptor in the `PackageDefinition`. The issue arises when we attempt multiple invocations of `protoLoader.load()` as each may only fetch a subset of the package (in this example from _a.proto_ in the first invocation and _b.proto_ in the second). In these cases we have multiple different _vendor.proto_ references which breaks the assumptions of the reflection specification in which files are often looked up by name.

## Implementation

I (jtimmons) will implement this once the maintainers have approved

## Open issues (if applicable)

### ~1. Package Publishing~

~Not sure how the package versioning and publishing works but may need some help from a maintainer to set that up and publish things into the `@grpc` scope~

**update**: addressed in [this PR thread](https://github.com/grpc/proposal/pull/397#discussion_r1355334943). Maintainers will handle package publishing separate from the initial implementation.

### ~2. keepCase Option~
~One of the more awkward pieces in here is dealing with grpc-js/proto-loader's `keepCase` option, though this isn't unique to this library. Consumers can have their gRPC server configured with `keepCase` either on or off which changes the message handling substantially: for example if enabled then we would need to refer to the `listServices` field as `list_services` instead.~

~I'm not sure if there's an official way of handling this but in the existing implementation we've had to introduce some [kludgey manual camel-casing logic](https://gitlab.com/jtimmons/nestjs-grpc-reflection-module/-/blob/30b67a78ff99e31ae54a0ab34c3784316579c665/src/grpc-reflection/controllers/v1.base.controller.ts#L48) to handle users of both settings.~

**update**: addressed in [this PR thread](https://github.com/grpc/proposal/pull/397#discussion_r1355335656). Non-issue and seems to be unique to how NestJS gRPC controllers work - will be opening a ticket in their repo to address this
