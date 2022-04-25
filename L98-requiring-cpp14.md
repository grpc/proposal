Requiring C++14 in gRPC Core/C++ Library
----
* Author(s): veblush
* Approver: a11r
* Status: Draft
* Implemented in: n/a
* Last updated: April 25, 2022
* Discussion at: https://groups.google.com/g/grpc-io/c/cpSVzf3rZYY

## Abstract

gRPC starts requiring C++14.

## Background

gRPC has been requiring C++11 from 2017 by
[Allow C++ in gRPC Core Library](L6-core-allow-cpp.md). As all compilers
that gRPC supports are now capable of C++14, gRPC is going to require
C++14 to benefit from new C++14 features.

## Proposal

gRPC 1.46 will be the last release supporting C++11, future releases will
require C++ >= 14. We plan to backport critical (P0) bugs and security fixes
to this release for a year, that is, until 2023-06-01.

This change won't bump the major version of gRPC since this doesn't introduce
API changes. Hence, the next version requiring C++14 will be 1.47.
