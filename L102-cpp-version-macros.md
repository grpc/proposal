L102: New gRPC C++ version macros
----
* Author(s): veblush
* Approver: markdroth
* Status: Approved
* Implemented in: gRPC C++
* Last updated: Oct 26, 2022
* Discussion at: https://groups.google.com/g/grpc-io/c/X2VsZ1MlySg

## Abstract

New public macros telling the version of gRPC C++ will be added to enable
libraries and applications using gRPC C++ to behave differently depending
on the gRPC C++ version at compile-time.

## Background

It's a widely used code pattern to behave differenly based on the version
information available at compile-time. Following is an example

```
#ifdef GRPC_CPP_VERSION_MAJOR
#  if GRPC_CPP_VERSION_MAJOR == 1
#    if GRPC_CPP_VERSION_MINOR >= 60
       // Use a new feature available from gRPC C++ 1.60
#    else
       // Do some workaround for gRPC C++ 1.59 or older
#    endif
#  else
       // New major version!
#  endif
#else
// Do some workaround for old gRPC C++
#endif
```

This has been asked by users (e.g [#25556](https://github.com/grpc/grpc/issues/25556)) and 
other Google OSS libraries (e.g. 
[Abseil](https://github.com/abseil/abseil-cpp/blob/8c0b94e793a66495e0b1f34a5eb26bd7dc672db0/absl/base/config.h#L88-L115), 
[Protobuf](https://github.com/protocolbuffers/protobuf/blob/0d0164feff22a4c9a3e884c60c2987ae87969957/src/google/protobuf/stubs/common.h#L82-L87),
and [Cloud C++](https://github.com/googleapis/google-cloud-cpp/blob/d33e46f94b2dfa6bcad0f2addfbfb5eb4978f40a/google/cloud/internal/version_info.h#L18-L20))
already provide version macros so it makes sense that gRPC has similar ones.

## Proposal

Following macros will be added to the `grpcpp.h` header file.

- `GRPC_CPP_VERSION_MAJOR`: Major version part (e.g. 1)
- `GRPC_CPP_VERSION_MINOR`: Minor version part (e.g. 46)
- `GRPC_CPP_VERSION_PATCH`: Patch version part (e.g. 1)
- `GRPC_CPP_VERSION_TAG`:  Tag version part (e.g. empty or rc0)
- `GRPC_CPP_VERSION_STRING`: Whole version string (1.46.1-rc0)

This changed is going to be reflected via https://github.com/grpc/grpc/pull/31033.
