gRPC C++ Public Header Directory Change
----
* Author(s): muxi
* Approver: nicolasnoble
* Status: approved
* Implemented in: C++
* Last updated: 01/25/2018
* Discussion at: https://groups.google.com/d/msg/grpc-io/SDyc0hSWWG8/pX2n9BSNAQAJ

## Abstract
This proposal is to change the name of directory `include/grpc++` to `include/grpcpp` for compatibility issues. 

## Background
gRPC C++ headers have been using the directory `include/grpc++` for a long time. Currently, users use `#include <grpc++/...>` to include gRPC headers in their source code. Some gRPC internal code and public headers also use this style to include gRPC public headers.

However, since the name `grpc++` has special character `+`, it causes compatibility issue when migrating to certain other platforms.

One of such example is Xcode. Some gRPC users, such as Firestore, need to build gRPC C++ library on iOS as Apple framework. The problem emerges on this platform due to 3 restrictions:
- gRPC public headers use `#include <grpc++/...>` to include gRPC C++ headers;
- An iOS app must include headers of a Framework with the format `#include <framework_name/path_to_header/header.h>`;
- Apple framework's name must be C99 extended identifier compatible; `grpc++` is not compatible.

The 3 restrictions make it impossible to make gRPC C++ library work as Apple framework. We believe this issue can happen somewhere else too in the future (e.g. [grpc#14089](https://github.com/grpc/grpc/pull/14089) is another example where this naming convention creates problem).

## Proposal

The proposal is to migrate `include/grpc++` directory to `include/grpcpp` in a backwards compatible manner. The objective is that both styles of inclusion `#include <grpc++/...>` and `#include <grpcpp/...>` can be used for all past and future gRPC C++ users.

### Changes to gRPC C++ API

- Make `include/grpcpp` directory the new location for all gRPC C++ public headers; move all current headers from `include/grpc++` to `include/grpcpp`; update all corresponding inclusions inside gRPC code base;
- Make wrapper headers in `include/grpc++` for all C++ public headers, preserving the directory structure. This makes current gRPC users build with old directory name. Add deprecation notice to wrapper headers as comments. 
- Update build systems to expose both headers and wrappers headers to users.

## Rationale
We think this change is reasonable since, as mentioned in Background section, the old directory name may hurt again in the future. Since this is a rather big API change in C++, wrapper headers are created to make current users build. With this change in place:
- Users who need to maintain backwards compatibility can keep using the old directory name;
- New users should use the new directory name;
- Users who need to use gRPC C++ as Apple framework must use the new directory name.

## Implementation
Will be implemented as part of the [gRPC C++ cocoapods library](https://github.com/grpc/grpc/issues/13582) efforts.
