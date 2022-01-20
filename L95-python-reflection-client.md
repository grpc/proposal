Python Reflection Client
----
* Author: Tomer Vromen
* Approver: lidizheng, gnossen
* Status: In Review
* Implemented in: Python
* Last updated: 2022-01-20
* Discussion at: https://groups.google.com/g/grpc-io/c/1fAWSxCy2mM

## Abstract

Add a programmatic way to access reflection descriptors from client-side Python.

## Background

Reflection is a means by which a server can describe which services and messages it supports.
The Reflection protocol is already part of the [gRPC repository](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md),
and there are server-side implementations of it in several languages, including C++ and Python - see [gRPC Python Server Reflection](https://github.com/grpc/grpc/blob/master/doc/python/server_reflection.md)

However, the only client-side implementation is in [C++](https://github.com/grpc/grpc/blob/master/doc/server_reflection_tutorial.md#use-server-reflection-in-a-c-client).
It seems that a client-side Python implementation is a missing to complete the picture.

This proposal closes this gap.

### Related Proposals: 

* Loosely related: [Promote the Reflection Service from v1alpha to v1](https://github.com/grpc/proposal/blob/master/A15-promote-reflection.md)
    - The proposal discusses the stability of the reflection service.
    - Accepted but not yet implemented - see [#27957](https://github.com/grpc/grpc/pull/27957).

## Proposal

Provide a Python implementation for client-side reflection, modeled after the existing C++ implementation.

* Implement `ProtoReflectionDescriptorDatabase` which implements the
[`DescriptorDatabase`](https://googleapis.dev/python/protobuf/latest/google/protobuf/descriptor_database.html#google.protobuf.descriptor_database.DescriptorDatabase)
interface.
* Write tests.
* Write documentation.

## Rationale

Python provides an easy interface for reflection, due to its dynamic nature.
For example, retrieving the descriptor for a service can be as simple as
```Python
service_desc = desc_pool.FindServiceByName("helloworld.Greeter")
```

The alternative is that anyone who wishes to have a Python reflection client will have to implement this by themselves.

Adding proper tests to the codebase ensures correctness even when (if) the reflection protocol changes.

The main downside is having a bigger API surface to support.

Related discussion: [how to get reflected services and rpc interface in grpc-python?](https://groups.google.com/g/grpc-io/c/SS9pkHMiLK4/m/OcwtakfqBQAJ)

## Implementation

Implementation is already available as a PR [#28443](https://github.com/grpc/grpc/pull/28443).

