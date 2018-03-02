L26: [C++] Make Serialization Mechanism Customizable Call
----
* Author(s): ncteisen
* Approver: vjpai
* Status: Draft
* Implemented in: n/a
* Last updated: 02/28/18
* Discussion at: n/a

## Abstract

This document proposes a change to [SerializationTraits](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/serialization_traits.h) in order to allow the serialization mechanism to be configurable per call. Most use cases will be unaffected, but this will make it much easier for user to inject custom serialization code into the gRPC pathway.

## Background

Currently, gRPC supports a mechanism for custom serialization. Users must implement the `grpc::SerializationTraits<Message, void>` class for their specific Message type. Forcing users to implement a class that exists in the grpc namespace is a form of internal API leakage. Additionally, this method does not give users the granularity to change the serialization system per-call, only at a per-type method.

### Serialization in Java and Go

Go already supports per-call serialization mechanisms. Users may implement serializes (Go calls them Codecs), and then set the Codec to be used at channel creation time (via DialOptions), or at call creation time. This is documented in detail [here](https://github.com/grpc/grpc-go/blob/master/Documentation/encoding.md).

Java also supports attaching custom marshallers to the method descriptor, as in [this example](https://github.com/grpc/grpc-java/tree/master/examples/src/main/java/io/grpc/examples/advanced). To get per-call granularity, a user can use an interceptor to manipulate the method descriptor for each call as in [this function](https://github.com/grpc/grpc-java/blob/master/android-interop-testing/app/src/main/java/io/grpc/android/integrationtest/InteropTester.java#L758).

### Related Proposals: 
* This related to [L25](https://github.com/grpc/proposal/pull/61), in that it is making it easier to customize serialization code.

## Proposal

We propose a new way to inject custom serialization code. Currently call objects are templated on request and response type. For example:

```C++
template <class W, class R>
class ClientAsyncReaderWriterInterface;
```

We will add templates for serialization classes for both the request and the response. These templates would default to the existing serialization traits objects. This example would look like:

```C++
template <class W, class R, 
          class WCodec = SerializationTraits<W, void>, 
          class RCoder = SerializationTraits<R, void>>
class ClientAsyncReaderWriterInterface;
```

In the default case, nothing will have to change. But this will give greater control to users who would like to supply custom serialization.


## Implementation

Not started yet.
