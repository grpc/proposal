Node Message Type Information
----
* Author(s): mlumish
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node
* Last updated: 2018-11-19
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This document proposes a generic format for representing message type information in package definition objects that are intended for use with gRPC, as well as a more specific format for representing Protobuf message and enum type information using that more general format. In addition, this document specifies how the `@grpc/proto-loader` library will expose this information.

## Background

In the original `grpc` library, when loading `.proto` files using `grpc.load` or `grpc.loadObject`, services are represented by a custom type defined for gRPC that is not specific to the Protobuf implementation or even the Protobuf format in general. However, enums and messages are represented using the reflection types specific to Protobuf.js 5. This restricted our ability to change the underlying Protobuf.js. We solved that problem by simply omitting that type information from the output of `@grpc/proto-loader`, but users have since requested that that information be made available in issues and pull requests such as grpc/grpc-node#407 and grpc/grpc-node#448. In addition, the Google Cloud client libraries need some of this reflection information to implement certain features.

## Proposal

### Generic type object structure

The generic object structure for representing message types will be as follows:

```ts
{
  format: string;
  type: any;
}
```

The `format` will be a string that describes the kind of message type information that is represented by the object. Examples of format strings that might be used include "JSON Schema" and "Protocol Buffer 3 DescriptorProto". The `type` will be an object or other value, depending on the format, that represents the specific message type. For example, if `format` is "JSON Schema", `type` might be a plain JavaScript object representation of the JSON Schema of the specific message type. These are the only two fields that will be assumed to have specific semantics across all possible formats. Some formats may define additional fields in the same object.

### Protobuf type object structure

The object structure for representing Protobuf message types will be as follows:

```ts
{
  format: "Protocol Buffer 3 DescriptorProto",
  type: DescriptorProtoObject,
  fileDescriptorProtos: Buffer[]
}
```

A `DescriptorProtoObject` is a plain JavaScript object that contains the [canonical JSON Mapping](https://developers.google.com/protocol-buffers/docs/proto3#json) representation of a `DescriptorProto` message defined in the well known proto file `descriptor.proto`. The `fileDescriptorProtos` array contains a list of `.proto` files that is sufficient to fully define this message type, represented as serialized `FileDescriptorProto` messages defined in the same proto file `descriptor.proto`. The primary purpose of this field is to be used in implementing the gRPC reflection API.

Protobuf enum types will be represented with a nearly identical structure, except with `EnumDescriptorProto` instead of `DescriptorProto`.

### `@grpc/proto-loader` type output

The output of `@grpc/proto-loader` will include Protobuf type objects in two different places:

 - The top-level object will include every loaded message and enum type in this object format referenced by its corresponding fully qualified name.
 - Each `MethodDefinition` object will contain the keys `RequestType` and `ResponseType` that will map to references to the corresponding message type objects.

## Rationale

gRPC itself does not depend on Protobuf or any other specific serialization format. So, for maximal generality, we need a representation that specifies the serialization format itself in data. Similarly, for generality across different Protobuf implementations, we need an implementation-independent standard representation. The `DescriptorProto` message format is an existing standard representation for Protobuf types, and the JSON representation is easy to use within JavaScript and is independent of the Protobuf implementation.

The proposed `@grpc/proto-loader` output matches how the type information is output when using `grpc.load`, but it uses the new generic type representation instead of the type representation specific to Protobuf.js 5.


## Implementation

I (murgatroid99) will implement this proposal in the `@grpc/proto-loader` library, using [Protobuf.js's descriptor proto compatibility extension library](https://github.com/dcodeIO/protobuf.js/tree/master/ext/descriptor) to transform the Protobuf information we are already loading into the generic `DescriptorProto` representation described above.

Further in the future, this should also be implemented in the Node gRPC `protoc` plugin distributed in the `grpc-tools` package.
