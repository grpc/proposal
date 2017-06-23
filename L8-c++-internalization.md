Separate internal-only and public sections of the C++ wrapping
----
* Author(s): vjpai
* Approver: ctiller
* Status: Approved
* Implemented in: C++
* Last updated: July 6, 2017
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/Z8S9P4NzGHs

## Abstract

Maintain separation of C++ classes and functions that are intended
to be directly used by application developers from those that are
supposed to be used only through codegen or for that exist for gRPC's
internal implementation (e.g., interactions with the core API).

## Background

Essentially all of gRPC C++ lives in `namespace grpc`, giving the
appearance that all portions are equally intended as part of the
publicly-exposed API. This gRFC aims to clarify this by categorizing
the gRPC C++ wrapping, moving some components to `namespace
grpc::internal`, and moving some class constructors to `private` (with
friends or appropriately-named `static` factories for internalized generation).

### Related Proposals: 

N/A

## Proposal

The C++ gRPC wrapping includes numerous classes and functions, but
they fall into five separate categories:

1. Documented for public use
1. Intended for interfacing with serialization layers such as protobuf
1. Intended for use through the code-generation layer
1. Intended for use in the internal implementation of gRPC C++
1. Intended for interfacing with gRPC core

This gRFC aims to clarify that the first two are intended for public
use (the second one to support custom serializers), but that the last
three should not be used as such. This will be achived by
moving classes and functions in the latter three categories to `namespace
grpc::internal`.

* Accessed through codegen 
  - `RpcMethod`
  - `BlockingUnaryCall`

* gRPC C++ implementation
  - `CallHook`
  - `CallOpSetInterface`
  - `CallOpSet`
  - `SneakyCallOpSet`
  - `CallNoOp`
  - `RpcMethodHandler`
  - `UnknownMethodHandler`
  - `ClientStreamingHandler`
  - `ServerStreamingHandler`
  - `TemplatedBidiStreamingHandler`
  - `BidiStreamingHandler`
  - `StreamedUnaryHandler`
  - `SplitServerStreamingHandler`

* gRPC C++ interface with core
  - `CompletionQueueTag`
  - `Call`
  - `CallOpSendInitialMetadata`
  - `CallOpSendMessage`
  - `CallOpClientSendClose`
  - `CallOpServerSendStatus`
  - `CallOpClientRecvStatus`
  - `CallOpRecvInitialMetadata`
  - `CallOpRecvMessage`
  - `CallOpGenericRecvMessage`
  - `MetadataMap`

Additionally, some classes in the first category are only supposed to
be created through the code generator or method handlers. To support
this, we will privatize their constructors and allow them to be
created only by internalized `friend` classes or through `static`
factories invoked by the code generator (which are themselves moved to
a new member `struct` called `struct internal` in each of the classes).

- `ClientReader`
- `ClientWriter`
- `ClientReaderWriter`
- `ServerReader`
- `ServerWriter`
- `ServerReaderWriter`

Whereas the code generator could previously use `new
::grpc::ClientReader<R>`, it will now use
`::grpc::ClientReader<R>::internal::Create` as a result of this
proposal. This is slightly bulkier but it makes it clear that public
code is not supposed to directly construct or `new` an object of type
`ClientReader`.

In some cases, the `static` factory already existed, but will now be
moved to a `struct internal` member class within the class.

- `ClientAsyncResponseReader`
- `ClientAsyncReader`
- `ClientAsyncWriter`
- `ClientAsyncReaderWriter`

The effect on the above classes is that the code generator will call
`::grpc::ClientAsyncResponseReader::internal::Create` instead of
`::grpc::ClientAsyncResponseReader::Create`. This change is much less
significant than the above.

Interface-only classes that relate to the first category will also be
moved to the `grpc::internal` namespace, except for the immediate
parents of final user-exposed classes (since those interfaces are
useful for creating mock versions).

- `AsyncReaderInterface`
- `AsyncWriterInterface`
- `ClientAsyncStreamingInterface`
- `ClientStreamingInterface`
- `ReaderInterface`
- `ServerAsyncStreamingInterface`
- `ServerStreamingInterface`
- `WriterInterface`

## Rationale

Despite the categories described above, the language does not provide
a direct mechanism for specifying this difference, so public users can
actually use any part of the wrapping that they choose. For example,
[Tensorflow](https://github.com/tensorflow/tensorflow/blob/r1.2/tensorflow/core/distributed_runtime/rpc/grpc_worker_service_impl.h)
bypasses the gRPC code generator and directly uses gRPC components in
each category except for core-interactions.

Although circumventing apparent abstraction layers can be a bid to
gain performance, it can also hurt achieved application performance by
locking applications into deprecated gRPC implementation mechanisms
instead of allowing the latest performance enhancements in gRPC C++.

The second category is remaining untouched because we have advertised
gRPC as accepting pluggable serializers, and this should remain
untouched. These include `SerializationTraits`, `Slice`, and other
related classes. Even though these are not used as a common part of
the documented public API, there are external users that reasonably
want these and we can continue supporting them.

## Implementation

This is implemented in C++ in pull-request
https://github.com/grpc/grpc/pull/11572. It is intended as a safe
build-only change. In fact, only two places in our own `test` suite
directly used internal classes and they have been resolved as follows:

1. `bm_cq`: completion queue processing benchmark. Since this is a
microbenchmark, allow it to access `internal::CompletionQueueTag`
1. QPS worker: this benchmark was directly using
`ReaderInterface` and `WriterInterface` knowing that they were the
parent-classes of various sub-classes that could be used in
the test. Instead, switch this to a `template` and use duck-typing
instead of inheritance.

Ongoing work will be to maintain discipline in deciding whether a
class or function belongs in `grpc` or `grpc::internal` as well as
deciding whether a class truly needs a public constructor.

## Open issues (if applicable)

This PR will cause problems for gRPC users that are using the
interfaces that are intended for internal use. This should not be a
large number of gRPC users since these APIs were not documented for
public use.
