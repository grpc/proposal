Title
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

Currently, gRPC supports a mechanism for custom serialization. Users must implement the `grpc::SerializationTraits<Message, void>` class for their specific Message type. Forcing users to implement a class that exists in the grpc namespace is a form of internal API leakage. Additionally, this method does not give users the granularity to change the serialization system per-call.

### Related Proposals: 
* This related to [L25](https://github.com/grpc/proposal/pull/61), in that it is making it easier to customize serialization code.

## Proposal

We propose a new way to inject custom serialization code. Currently call objects are templated request and response type. For example:

```C++
template <class W, class R>
class ClientAsyncReaderWriterInterface;
```

We will add templates for serialization classes for both the request and the response. These templates would default to the existing serialization traits objects. This would look like:

```C++
template <class W, class R, 
          class WCodec = SerializationTraits<W, void>, 
          class RCoder = SerializationTraits<R, void>>
class ClientAsyncReaderWriterInterface;
```

In the default case, nothing will have to change. But this will give greater control to users who would like to supply custom serialization.

## Implementation

Not started yet.
