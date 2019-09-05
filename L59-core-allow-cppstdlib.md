Allow C++ standard library in gRPC Core Library
----
* Author(s): veblush
* Approver: a11r
* Status: Draft
* Implemented in: n/a
* Last updated: September 4, 2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/umQyyXmNr2w

## Abstract

Allow C++ Standard Library to be used in the gRPC Core library.

## Background

In the past, gRPC team allowed the use of C++ code in the gRPC Core
[library](L6-core-allow-cpp.md), but we stayed conservative and avoided
introducing a dependency on C++ standard library.
Being able to use C++ has lead to increased productivity, but there are
still significant limitations, because many useful C++ features depend on
the standard C++ library. Not using C++ standard library also prevents us
from using other C++ libraries, like Abseil, which rely on the C++ standard
library.

To minimize duplication of code efforts, and to increase productivity,
gRPC team proposes to allow developers to use the C++ standard library
within the gRPC Core library.

### Related Proposals:

* [Allow C++ in gRPC Core Library](L6-core-allow-cpp.md)

## Proposal

gRPC team proposes to allow C++11 standard library usage in gRPC Core.
The previous C++ usage rules will be valid except for the restriction on
C++ features and the header-only library.

- All C++11 features may be used. The following C++11 features will become
  available once this proposal is approved
  - `new` and `delete`
    - All built-in std allocators can be ready to use, without having to
      specify custom new/delete functors when special behavior is not needed.
    - Even though `new` and `delete` will be available for use, this doesn't
      mean that all code should use this feature. Where `gpr_malloc` and
      `gpr_free` are used intentionally, `grpc_core::New` and
      `grpc_core::Delete` need to be used.
  - Pure virtual functions:
    Regular form `= 0` can be used instead of `GRPC_ABSTRACT`
- All features from the C++11 standard library may be used.
  Previously, only header-only features were allowed.

#### Caveats

There are caveats, since the gRPC-wrapped library is distributed as a
binary form. The wrapped library includes a gRPC Core artifact, which
affects all wrapped libraries. The C++ standard library should be installed
to make the wrapped gRPC library work properly.

The goal is to try not to ask users to install an additional C++ standard
library. The solution varies depending on the platform.

 - Linux: Uses the same restriction from
    [manylinux1](https://www.python.org/dev/peps/pep-0513) policy
    because it targets Linux released after 2007.
 - Windows: Since Windows hasn't been bundled with the C++ library,
    gRPC has been linked to C++ standard library in a static way.
    So, this change doesn't affect Windows.
 - MacOS/iOS/Android: Since a specific version of the C++ library
    has been distributed on these platforms,
    users don't need to worry about this.

Simply speaking, gRPC contributors can use C++11 library features that
are available in manylinux1. For details on manylinux, read
[PEP 513](https://www.python.org/dev/peps/pep-0513)
Here is what matters to C++:

  - GLIBCXX <= 3.4.9 (GCC 4.2.0)

Most of the C++ standard library is available on this restriction.
Following is a partial list of missing features due to `manylinux1`
restrictions:

  - std::chrono
  - std::numeric_limits for long long and some methods
  - string and stream support for wchar_t
  - new implementation of string and list compliant with C++11
    (still old implementation of string and list are available.
    for details on this, read [Dual ABI](
    https://gcc.gnu.org/onlinedocs/gcc-9.2.0/libstdc++/manual/manual/using_dual_abi.html))

## Rationale

Using the C++ standard library provides these advantages:
- gRPC Core can use all useful and essential classes in the C++ standard
  library, which minimizes the need to reinvent code.
- gRPC Core can use other libraries that require the C++ standard library.
  Previously, linking to Abseil or protobuf was impossible because it
  required the C++ standard library.
  With this proposal, those can be used.

All previous restrictions on linking the C++ standard library are arbitrary
and can change anytime because they are implementation-specific, and aren’t
part of a standard or specification. Having gRPC built by `gcc` and `libstdc++`
doesn't guarantee that it can achieve the same goal with `clang` and `libc++`.

## Implementation

1. Change the build configuration for all wrapped language to add the dependency
   to the C++ standard library.
   [#19918](https://github.com/grpc/grpc/pull/19918)
2. Make changes to use the C++ standard library to prove it's working.
   The change should be small enough so that we can easily roll it back.
   [#20059](https://github.com/grpc/grpc/pull/20059)
   - Replace `grpc_core::map` with `std::map`
   - Replace `GRPC_ABSTRACT` with `= 0`
3. Add new build tests to make sure that the version of library gRPC uses is
   old enough for most platforms to have it.
   - For Python and Ruby, manylinux1 is a good candidate to target.

## Schedule

1. New dependency against the C++ standard library is added with some code
   depending on it. This must be easy to roll back in case of errors.
   - gRPC master branch gets this change with
      [#19918](https://github.com/grpc/grpc/pull/19918) (on September 3, 2019)
   - Released with gRPC 1.24 (scheduled on September 24, 2019)
2. Once gRPC 1.24 release looks fine like there is no breaking change,
   the gRPC Core will be able to use the C++ standard library.
   - From gRPC 1.25-dev (after September 24, 2019)
