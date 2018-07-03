L26: [C++] Add Raw API to C++ Server-Side Generated Code
----
* Author(s): ncteisen
* Approver: vjpai
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/15771
* Last updated: 2018-07-01
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/3FTSA67iios

## Abstract

This document proposes adding a new API to the C++ generated code layer for servers. This API would designate that the selected methods be handled in a _raw_ manner, which means that their signatures will be written in terms of `::grpc::ByteBuffer` as opposed to API specific protobuf objects.

## Background

Many teams have expressed interest in having a proto defined service that has some specialized methods that do custom serialization. The need for this use case has caused pain for users. Notably, TensorFlow was forced check in generated code in [this commit](https://github.com/tensorflow/tensorflow/commit/6ba4995be6372d1ca5e01eae649c1d8750b65857#diff-7d3228bae4cf4c12f11c17eb147fe5ba). 

### Related Proposals: 
* This related to [L25](https://github.com/grpc/proposal/pull/61), in that it is making it easier to customize serialization code.

## Proposal

We will add a new API to the generated server code that signals for a method to be written in terms of `::grpc::ByteBuffer`. Marking a method as `Raw` naturally means that it is asynchronous.

Selecting which methods are `Raw` will be handled in the same manner as we currently allow for selection of Generic, Async, or Streamed methods. For example:

```C++
using SpecializedServer = 
    grpc::TestService::WithRawMethod_Foo<grpc::TestService::AsyncService>
```

This server would using protobuf to negotiate all methods _except_ Foo, which would be handled with ByteBuffers. This allows the application to use whatever serialization mechanism they want for Foo.

Marking a method as `Raw` is type unsafe since gRPC library cannot ensure that the user is serializing and deserializing with the same protocol. This feature is meant for "power users" who are willing to enforce the serialization invariant in their code.

Continuing with the example, if Foo were unary, then the server would interact with it like so:

```C++
// setup
SpecializedServer* server = BuildServer(...)
grpc::ServerCompletionQueue cq;
grpc::ServerContext srv_ctx;

// incoming
grpc::ByteBuffer recv_request_buffer;
grpc::GenericServerAsyncResponseWriter response_writer(&srv_ctx);
service->RequestEcho(&srv_ctx, &recv_request_buffer, &response_writer,
                     &cq, &cq, tag(1));

// outgoing
grpc::ByteBuffer send_response_buffer = ProcessRequestBB(recv_request_buffer);
response_writer.Finish(send_response_buffer, Status::OK, tag(2));
```

All other arities follow the same pattern.

## Implementation

The implementation will follow the same pattern as the generated code for async API, but will be written with ByteBuffer instead of proto objects. This implementation will cause the `final` qualifier to be removed from the currently generated `WithAsyncMethod_Foo` classes.

Implementation is in [#15771](https://github.com/grpc/grpc/pull/15771)
