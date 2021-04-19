C++ API changes on ByteBuffer and Slice
----
* Author(s): 
* Approver: vjpai
* Status: Draft
* Implemented in: C++
* Last updated: 2021-04-19
* Discussion at: TBD

## Abstract

For the better integration between gRPC and FlatBuffers, additional methods will be added to `grpc::ByteBuffer` and `grpc::Slice` allowing FlatBuffers to access data without copying data.

## Background

FlatBuffers has been using gRPC core methods to exchange data with gRPC but it's not ideal because gRPC core doesn't promise API stability, which caused it broken time to time.
To address this problem, FlatBuffers needs to use public gRPC C++ API which will be newly added to provide it more control on memory management of `grpc::ByteBuffer` and `grpc::Slice` so that it can get more stability without performance impact.

Related issues:
- https://github.com/grpc/grpc/issues/20594
- https://github.com/google/flatbuffers/issues/5836

### Related Proposals: 

N/A

## Proposal

### grpc::ByteBuffer

`grpc::ByteBuffer` is going to have two additional methods. `TrySingleSlice` will returns a single `grpc::Slice` if it's made up with an uncompressed slice like `absl::Cord::TryFlat` ([code](https://github.com/abseil/abseil-cpp/blob/732c6540c19610d2653ce73c09eb6cb66da15f42/absl/strings/cord.h#L639)). This is useful when accessing data without copying data via the returned slice. Otherwise it fails. 
Another method, `DumpToSingleSlice` returns a newly created slice containing whole data by copying underlying slices into a single one. This will simply how to read data from `grpc::ByteBuffer`.

```
Status TrySingleSlice(Slice* slice) const;
Status DumpToSingleSlice(Slice* slice) const;
```

### grpc::Slice

`grpc::Slice` is going to have one simple method, `sub` returning a sub-slice of given ByteBuffer and the span.

```
Slice sub(size_t begin, size_t end) const;
```

## Rationale

The list of APIs to be added here is based on the requirement of Flatbuffers to reimplement its plugin using gRPC C++ APIs only. These APIs are already in `absl::Cord` and `absl::string_view` so these are considered acceptable to have in gRPC as well.

## Implementation

Implementation will be done by https://github.com/grpc/grpc/pull/26014
