L25: [C++] Make GrpcProtoBuffer{Reader|Writer} Public
----
* Author(s): ncteisen
* Approver: vjpai
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/14541
* Last updated: 2018-03-01
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/3FTSA67iios

## Abstract

We propose to move two classes, `GrpcProtoBufferWriter` and `GrpcProtoBufferReader`, to a public directory, so that users may create custom serialization traits without having to "reinvent the wheel".

## Background

We have seen several user complaints that creating custom serializers is not an easy task. For example, TensorFlow [duplicated an internal file](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/core/distributed_runtime/rpc/grpc_serialization_traits.h), which led to a [nasty bug](https://github.com/grpc/grpc/issues/10161). Another project had to use a hacky workaround to subtype the `GrpcProtoBuffer{Reader|Writer}` classes in order to implement a zero copy serializer.

### Related Proposals: 
* This relates to [L26](https://github.com/grpc/proposal/pull/63), in that it is making it easier to customize serialization code.

## Proposal

We will rewrite `GrpcProtoBuffer{Reader|Writer}` in terms of the public class, `grpc::ByteBuffer`. Then we will move `GrpcProtoBuffer{Reader|Writer}` out of the internal namespace, and add header files in `include/grpcpp/support` so that the classes become accessible.

## Implementation

Implementation is in [#14541](https://github.com/grpc/grpc/pull/14541)
