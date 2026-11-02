Node.js gRPC 2.0 API changes
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node.js
* Last updated: 2017-11-27
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/AIJ9oGXKCrU

## Abstract

A list of changes to be made to the Node.js gRPC API as part of the update to version 2.0.

## Background

We need to increment the major version of the Node.js gRPC library in order to upgrade the Protobuf.js dependency, and to add a capability system to handle the differences between the C-based implementation and the new pure JavaScript implementation. We expect this to be the only major version bump for years, so this is the only opportunity for a while to make API-breaking changes.

## Proposal

### Protobuf.js-related Changes

 - Create a new separate package `@grpc/protobufjs` that exposes the `grpc.load` and `grpc.loadObject` functionality.
 - Upgrade the dependency on Protobuf.js from 5.0 to 6.8
 - Drop support for loading objects created by Protobuf.js 5
 - Modify the `load` and `loadObject` functions to generate implementation-agnostic service definition objects instead of implementation-specific client class objects.
   - Generation of client class objects from service definition objects would be part of the main gRPC API
   - A helper function can be added to the main gRPC API to load a namespace containing service objects and return a namespace containing client classes, which would be similar to the current output of `grpc.load` and `grpc.loadObject`
 - Use only the original method names from proto files in client and server objects. Possibly add an option to generate camel case names instead. Background:
   - The original design had the method names converted to camelCase, to match common JavaScript styles
   - As of gRPC 1.7, aliases have been added to both clients and servers to allow users to use the original names as written in the proto files instead of the modified names

### Capability-related changes

 - Add functionality to enable features if they are available (to be described in another gRFC).
 - Throw an error upon any access to non-enabled features.
   - This may require a change in how channel arguments are handled to reject unrecognized argument names.

### Remove Deprecated APIs

 - Remove the option `deprecatedArgumentOrder` from `grpc.load`, `grpc.loadObject`, and `grpc.makeGenericClientConstructor`.
 - Remove `grpc.closeClient`, `grpc.waitForClientReady`, and `grpc.getClientChannel` in favor of `grpc.Client.prototype.close`, `grpc.Client.prototype.waitForReady` and `grpc.Client.prototype.getChannel` respectively.

### Improve Credentials APIs

 - Divide the current `credentials` module API functions to explicitly expose the classes `CallCredentials` and `ChannelCredentials`. In particular:
   - `CallCredentials` will have the functions `createFromMetadataGenerator` and `createFromGoogleCredential`
   - `ChannelCredentials` will have the functions `createInsecure` and `createSsl`.
 - Update the `checkClientCertificate` argument of `ServerCredentials.createSsl` from a boolean to an enum to reflect the internal API.
 - Change `ChannelCredentials.createSsl` and `ServerCredentials.createSsl` to use objects instead of positional arguments. In particular, they should use options that are compaitible with [`tls.createSecureContext`](https://nodejs.org/api/tls.html#tls_tls_createsecurecontext_options) accepts: `key`, `cert`, and `ca`. `ServerCredentials.createSsl` will additionally accept a `checkClientCertificate` option.

### Improve Server APIs

 - Replace `tryShutdown` and `forceShutdown` functions with a single `shutdown` function with an additional `options` arguments that accepts a `graceful` option. With `graceful` set to `true`, the behavior will be identical to the current `tryShutdown` behavior, and with `graceful` set to `false`, the behavior will be identical to the current `forceShutdown` behavior. In both cases, it will accept an optional callback.

### Channels

 - Expose the `Channel` constructor as part of the API, with the same arguments the the Client constructor currently has.
 - Add alternate client constructor that accepts a channel instead of arguments to the channel constructor.

## Implementation

I (murgatroid99) will implement this during Q1 2018.
