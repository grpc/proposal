C++ API changes on ByteBuffer and Slice
----
* Author(s): 
* Approver: vjpai
* Status: Approved
* Implemented in: C++
* Last updated: 2021-04-19
* Discussion at: https://groups.google.com/g/grpc-io/c/OsAYW1mDJ9w

## Abstract

To better support custom serialization protocols besides protobuf, additional methods will be added to `grpc::ByteBuffer` and `grpc::Slice` those serialization protocols to access data without copying data.
This proposal purely addes new methods so there shouldn't be any breaking changes.

## Background

FlatBuffers has been using gRPC core methods to exchange data with gRPC but it's not ideal because gRPC core doesn't promise API stability, which broke it from time to time.
To address this problem, FlatBuffers needs to use public gRPC C++ API which will be newly added to provide it more control on memory management of `grpc::ByteBuffer` and `grpc::Slice` so that it can get more stability without performance impact.
We believe this approach will also be useful for other user-defined SerializationTraits.

Related issues:
- https://github.com/grpc/grpc/issues/20594
- https://github.com/google/flatbuffers/issues/5836

### Related Proposals: 

N/A

## Proposal

### grpc::ByteBuffer

`grpc::ByteBuffer` is going to have two additional methods. `TrySingleSlice` will returns a single `grpc::Slice` if it's made up with an uncompressed slice like `absl::Cord::TryFlat` ([code](https://github.com/abseil/abseil-cpp/blob/732c6540c19610d2653ce73c09eb6cb66da15f42/absl/strings/cord.h#L639)). This is useful when accessing data without copying data via the returned slice. Otherwise it fails. 
Another method, `DumpToSingleSlice` returns a newly created slice containing whole data by copying underlying slices into a single one. This will simplify how to read data from `grpc::ByteBuffer`.

```
Status TrySingleSlice(Slice* slice) const;
Status DumpToSingleSlice(Slice* slice) const;
```

### grpc::Slice

`grpc::Slice` is going to have one simple method, `sub` returning a sub-slice of given Slice and the span.

```
Slice sub(size_t begin, size_t end) const;
```

## Rationale

The list of APIs to be added here is based on the requirement of Flatbuffers to reimplement its plugin using gRPC C++ APIs only. These APIs are already in `absl::Cord` and `absl::string_view` so these are considered acceptable to have in gRPC as well.

## Implementation

Implementation will be done by https://github.com/grpc/grpc/pull/26014 and followings are excerpts from the PR to give an idea on how those functions will be implemented.

### grpc::ByteBuffer

```
Status ByteBuffer::TrySingleSlice(Slice* slice) const {
  if (!buffer_) {
    return Status(StatusCode::FAILED_PRECONDITION, "Buffer not initialized");
  }
  if ((buffer_->type == GRPC_BB_RAW) &&
      (buffer_->data.raw.compression == GRPC_COMPRESS_NONE) &&
      (buffer_->data.raw.slice_buffer.count == 1)) {
    grpc_slice internal_slice = buffer_->data.raw.slice_buffer.slices[0];
    *slice = Slice(internal_slice, Slice::ADD_REF);
    return Status::OK;
  } else {
    return Status(StatusCode::FAILED_PRECONDITION,
                  "Buffer isn't made up of a single uncompressed slice.");
  }
}

Status ByteBuffer::DumpToSingleSlice(Slice* slice) const {
  if (!buffer_) {
    return Status(StatusCode::FAILED_PRECONDITION, "Buffer not initialized");
  }
  grpc_byte_buffer_reader reader;
  if (!grpc_byte_buffer_reader_init(&reader, buffer_)) {
    return Status(StatusCode::INTERNAL,
                  "Couldn't initialize byte buffer reader");
  }
  grpc_slice s = grpc_byte_buffer_reader_readall(&reader);
  *slice = Slice(s, Slice::STEAL_REF);
  grpc_byte_buffer_reader_destroy(&reader);
  return Status::OK;
}
```

### grpc::Slice

```
/// Returns a substring of the `slice` as another slice.
Slice Slice::sub(size_t begin, size_t end) const {
return Slice(g_core_codegen_interface->grpc_slice_sub(slice_, begin, end),
                STEAL_REF);
}
```
