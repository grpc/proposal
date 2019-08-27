Allow C++ standard library in gRPC Core Library
----
* Author(s): veblush
* Approver: a11r
* Status: Draft
* Implemented in: n/a
* Last updated: August 19, 2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/umQyyXmNr2w

## Abstract

Allow C++ Standard Library to be used in the gRPC Core library.

## Background

Since C++ was allowed to write gRPC, more and more code has been written in C++.
But most features from C++ standard libraries were not allowed to use because
we didn't want to introduce new dependency against C++ standard library.

The decision to use C++ has been helping us to be more productive but it's not
best because we cannot use C++ standard library and a few of C++ features which
depend on this. This also prevents us from using other C++ library which rely 
on the library. (e.g. abseil)

This gRFC proposes allowing C++ standard library to be used within the gRPC 
Core library. It may free us from reinventing wheels which already exists
in the standard library and help us become more productive.

### Related Proposals:

* [Allow C++ in gRPC Core Library](L6-core-allow-cpp.md)

## Proposal

Allow C++11 standard library usage in gRPC Core. The previous C++ usage rule
will be valid except the restriction on C++ features and header-only library.

- All C++11 features are allowed to use. Followings were not possible to use
  but it becomes available with this.
  - `new` and `delete`
    - Even though `new` and `delete` now become possible to use, it doesn't
      necessarily mean that all code should use.
      For the place where `gpr_malloc` and `gpr_free` should be used
      intentionally, `grpc_core::New` and `grpc_core::Delete` need to be
      used.
  - Pure virtual functions.
    Regular form `= 0` can be used instead of `GRPC_ABSTRACT`

- All features from C++11 standard library are allowed to use.
  Previously only header-only features were allowed to use.

There are caveats since gRPC wrapped library are being distributed as a binary
form. Wrapped library includes a gRPC Core artifact so this change affects
all wrapped libraries and C++ standard library should be installed to make
wrapped gRPC library work properly.

Goal is to try not to ask users to install additional C++ standard library
if possible. Solution varies depending on each platform.

 - Linux: Uses the same restriction from 
    [manylinux1](https://www.python.org/dev/peps/pep-0513) policy
    because it targets old enough Linux.
 - Windows: Since Windows hasn't been bundled with C++ library,
    gRPC has been linked to C++ standard library in a static way.
    So this change doesn't affect Windows.
 - MacOS/iOS/Android: Since a specific version of C++ library has been
    distributed on these platforms, 
    users don't need to worry about this.

Simply speaking, we can use C++11 library features which are available
in `manylinux1`. For details on `manylinux`, you can see the 
[doc](https://www.python.org/dev/peps/pep-0513).
Here is what matters to C++.

  - GLIBCXX <= 3.4.9 (GCC 4.2.0)

Fortunately, most of C++ standrad library availble on this restriction.
Following is a partial list of missing features because of `manylinux1`.
  - std::chrono.
  - std::numeric_limits for long long and some methods.
  - string and stream support for wchar_t.
  - new implementation of string and list compliant with C++11.
    (still old implementation of string and list are available)

## Rationale

Using C++ standard library gives us several advantages:
- gRPC Core can use all useful and essential classes in C++ standard library.
  This allows us not to reinvent everything which are already available.
- gRPC Core can use other libraries which require C++ standard library.
  Previously linking to abseil or protobuf was impossible because it requires C++
  standard library. With this, we can consider it.

All previous restrictions not to link the C++ standard library are arbitrary
and it can change anytime because it's not part of standard or specification
but implementation details. Having gRPC built by `gcc` and `libstdc++` doesn't
guarantee that it can achieve the same goal with `clang` and `libc++`.

## Implementation

1. Change build configuration for all wrapped language to add the dependency
   to C++ standard library.
2. Make changes to use C++ standard library to prove it's working.
   At the same time, it should be small enough so that they are easily
   rolled back just in case
   - Replace `grpc_core::map` with `std::map`
   - Replace `GRPC_ABSTRACT` with `= 0`
3. Add new build tests to make sure that the version of library gRPC uses
   is old enough for most platforms to have it.
   - For Python and Ruby, manylinux1 is a good candidate to target.

## Schedule

1. New dependency against the C++ standard library is added with
   some code depending it. This has to be easily rolled back just in case.
   -> Target gRPC 1.25 (scheduled on October 22, 2019)
2. Once this version looks fine, all gRPC Core now is allowed to use the library.
   -> From gRPC 1.26-dev

## Open issues (if applicable)

- Platform reach may decrease, but we feel this will not be significant.
