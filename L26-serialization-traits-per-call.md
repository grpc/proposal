L26: [C++] Make Serialization Mechanism Customizable Call
----
* Author(s): ncteisen
* Approver: vjpai
* Status: Draft
* Implemented in: n/a
* Last updated: 2018-03-01
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

## Deeper Examples New API 

After this change, the C++ Async Streaming API would look like:

```C++
template <class R, class RCodec = SerializationTraits<R, void>>
class AsyncReaderInterface {
 public:
  virtual void Read(R* msg, void* tag) = 0;
};

template <class W, class WCodec = SerializationTraits<W, void>>
class AsyncWriterInterface {
 public:
  virtual void Write(const W& msg, void* tag) = 0;
  virtual void Write(const W& msg, WriteOptions options, void* tag) = 0;
  void WriteLast(const W& msg, WriteOptions options, void* tag) = 0;
};

template <class R, class RCodec = SerializationTraits<R, void>>
class ClientAsyncReaderInterface
    : public internal::ClientAsyncStreamingInterface,
      public internal::AsyncReaderInterface<R, RCodec> {};

template <class R, class RCodec = SerializationTraits<R, void>>
class ClientAsyncReaderFactory {
 public:
  template <class W, class WCodec = SerializationTraits<W, void>>
  static ClientAsyncReader<R, RCodec>* Create(ChannelInterface* channel,
                                      CompletionQueue* cq,
                                      const ::grpc::internal::RpcMethod& method,
                                      ClientContext* context, const W& request,
                                      bool start, void* tag);
};

template <class R, class RCodec = SerializationTraits<R, void>>
class ClientAsyncReader final : public ClientAsyncReaderInterface<R, RCodec> {
  void Read(R* msg, void* tag) override;

 private:
  friend class internal::ClientAsyncReaderFactory<R>;
  template <class W, class WCodec = SerializationTraits<W, void>>
  ClientAsyncReader(::grpc::internal::Call call, ClientContext* context,
                    const W& request, bool start, void* tag);
};
```

The Sync Unary API would look like:

```C++
template <class InputMessage, class OutputMessage, 
          class InputCodec, class OutputCodec>
Status BlockingUnaryCall(ChannelInterface* channel, const RpcMethod& method,
                         ClientContext* context, const InputMessage& request,
                         OutputMessage* result);
```


## Implementation

Not started yet.
