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

export interface ReflectionServer {
  constructor(pkg: PackageDefinition);
  reload(pkg: PackageDefinition);
  addToServer(server: MinimalGrpcServer);
}
```

this class will be responsible for managing requests for the various published versions of the gRPC Reflection Specification. At the time of writing, this includes `v1` and `v1alpha` but may include more in the future. These version-specific handlers will be isolated into their own services, such as the following which can be used for both `v1` and `v1alpha` due to their identical interface:

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

reflection.reload({ ... }) // can reload later on if additional services are added
```

## Rationale

### 1. Design Decision: use of proto-loader over protoc
several reflection implementations linked above leverage protoc in order to generate a representation of the proto schema to expose on the API. In this document we propose the use of proto-loader to inspect the schema at runtime instead in order to simplify the developer experience and be consistent with the [design of the `grpc-health-check` library](https://github.com/grpc/proposal/blob/ee75a4010214ddda02ba992e69f1c57be7f71497/L106-node-heath-check-library.md#switch-from-protoc-to-grpcproto-loader).

### 2. Design Decision: support multiple reflection implementations
currently not all reflection clients request the `v1` version of the spec so we need to include handlers for both `v1` and `v1alpha` to support both during the transition. For this reason we separate the reflection handling logic itself to allow for reuse across multiple service versions

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
