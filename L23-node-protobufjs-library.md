Standalone Node.js gRPC+Protobuf.js Library API
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: Ready for Implementation
* Implemented in: Node.js
* Last updated: 2022-09-20
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/t7XFE7Wevow

## Abstract

This proposes an API for the new standalone gRPC+Protobuf.js library that we will introduce.

## Background

The Node gRPC library currently depends on Protobuf.js version 5. Since Protobuf.js 6 was introduced, Protobuf.js 5 has not been maintained. In the past we have tried to upgrade that dependency in place, but that caused breaking changes that we did not accept. Our solution to this problem is to publish a new library that is solely responsible for loading `.proto` files using Protobuf.js and transforming them into a generic format that gRPC can use. The introduction of this new library provides an opportunity to improve the API for loading files based on feedback on the existing library.

## Proposal

For the purpose of definiing services in general, the existing gRPC library has defined the types [`MethodDefinition`](https://grpc.io/grpc/node/grpc.html#~MethodDefinition__anchor) and [`ServiceDefinition`](https://grpc.io/grpc/node/grpc.html#~ServiceDefinition__anchor). A `MethodDefinition` is an object with the following properties:

 - `path`: The URL path for accessing this method
 - `requestStream`: A boolean indicating whether there is a stream of request messages instead of a single message.
 - `responseStreams`: A boolean indicating whether there is a stream of response messages instead of a single message.
 - `requestSerialize`: A function for serializing a request object to a `Buffer`
 - `responseSerialize`: A function for serializing a response object to a `Buffer`
 - `requestDeserialize`: A function for deserializing a request object from a `Buffer`
 - `responseDeserialize`: A function for deserializing a response object from a `Buffer`
 - `options`: Raw method options object
 - `parsedOptions`: Method options properly parsed into an object

This information is sufficient to implement a client or service handler for any kind of method. Then a `ServiceDefintion` is just an object that maps method names to `MethodDefinition` objects. When loading a `.proto` file, more than one service may be loaded, so we additionally define a wrapping type `PackageDefinition`, which maps fully qualified service names to `ServiceDefinition` objects. For example, if a `.proto` file defines the package `a.b` with the services `Service1` and `Service2`, the resulting `PackageDefinition` would have the keys `"a.b.Service1"` and `"a.b.Service2"`.

The library exposes the following function:

```js
/**
 * Asynchronously load a .proto file and all of its transitive imports
 * @param {string|string[]} filename File path to the .proto file(s) to load.
 * @param {Object} options Options for loading the file and setting up the deserializers
 * @return {Promise<PackageDefinition>} A promise for an object containing definitions for all of the loaded services
 */
load(filename, options)

/**
 * Synchronously load a .proto file and all of its transitive imports
 * @param {string|string[]} filename File path to the .proto file(s) to load.
 * @param {Object} options Options for loading the file and setting up the deserializers
 * @return {PackageDefinition} A promise for an object containing definitions for all of the loaded services
 */
loadSync(filename, options)
```

The `filename` parameter can be either a string or an array of strings. If it's a string, it contains a path to a `.proto` file to load. If it's an array of strings, each element is a path to a `.proto` file to load, and the resulting `PackageDefinition` will contain services and methods from all `.proto` files in the array. This is the same behavior as in Protobuf.js's [`Root.load`](https://github.com/protobufjs/protobuf.js/blob/c72c752352347555406bafd7121acaed240fbf23/src/root.js#L190-L191).

The `options` parameter of those functions is the union of Protobuf.js's [`IParseOptions`](https://github.com/dcodeIO/protobuf.js/blob/cf7b26789f310dccf4c047c2e8ef5a3854f7f41e/index.d.ts#L1014) (which modifies how files are loaded) and [`IConversionOptions`](https://github.com/dcodeIO/protobuf.js/blob/cf7b26789f310dccf4c047c2e8ef5a3854f7f41e/index.d.ts#L1632) (which modifies deserialization functions), plus an `include` option to specify directories to search for dependencies. The full set of options is as follows:

 - `keepCase`: a boolean indicating that field names should be preserved. The default is to change them to camel case.
 - `longs`: Can be set to `Number` or `String` to indicate that long values should be represented as that type. The default is `Number`, or a safer object `Long` type if the library is installed.
 - `enums`: Can be set to `String` have enums represented as strings. The default is to use their numeric value.
 - `bytes`: Can be set to `Array` or `String` to indicate that bytes values should be represented as that type. The default is to use the Node built in type `Buffer`.
 - `defaults`: a boolean indicating that default values should be set on output objects.
 - `arrays`: a boolean indicating that empty arrays should be set for missing array values even if `defaults` is `false`.
 - `objects`: a boolean indicating that empty objects should be set for missing object values even if `defaults` is `false`.
 - `oneofs`: a boolean indicating that virtual oneof properties should be set to the present field's name.
 - `json`: a boolean indicating that additional JSON compatibility conversions should be performed.
 - `includeDirs`: an array of strings listing paths in which to search for imported `.proto` files.

## Rationale

This package provides very similar functionality to the existing `grpc.load` API, but with some differences. First, this API directly exposes all of the relevant Protobuf.js options instead of defining a subset of custom options. This package is explicitly a wrapper around Protobuf.js, so it makes more sense here to directly and consistently expose the relevant options from that library. Second, this API does not provide an equivalent to the `loadObject` function in the existing gRPC library. With the Protobuf.js options directly exposed, there is less benefit in allowing the user to separately load `.proto` files. Third, this adds an asynchronous API for loading `.proto` files. This has been requested before, and more closely matches existing Node APIs. It uses promises instead of callbacks because they are more versatile, and because Node is increasingly supporting them.


## Implementation

I (murgatroid99) will implement this during Q1 2018.
