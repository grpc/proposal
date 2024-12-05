L120: Requiring C++17 in gRPC Core/C++ Library
----
* Author(s): veblush
* Approver: markdroth
* Status: Draft
* Implemented in: n/a
* Last updated: Dec 4, 2024
* Discussion at: https://groups.google.com/g/grpc-io/c/HXnIJJnMdgc

## Abstract

To align with [the OSS Foundational C++ support policy](https://opensource.google/documentation/policies/cplusplus-support), gRPC is updating its minimum required C++ standard to C++17.

## Background

To leverage the advancements in C++ standards, gRPC has progressively updated its requirements.  
Initially, it adopted C++11 in 2017 (as per [L6: Allow C++ in gRPC Core Library](L6-core-allow-cpp.md)).
Then, in 2022, it transitioned to C++14 (as per [L98: Requiring C++14 in gRPC Core/C++ Library](L98-requiring-cpp14.md)).

Now, to align with the [the OSS Foundational C++ support policy](https://opensource.google/documentation/policies/cplusplus-support)
and stay consistent with its major dependencies (Abseil, BoringSSL, and Protobuf), gRPC is transitioning to require C++17.

## Proposal

gRPC 1.69 will be the final release compatible with C++14. Going forward, gRPC will require C++17. This change will take effect with gRPC 1.70.

gRPC 1.69 will continue to receive critical bug fixes (P0) and security updates for one year (until December 10, 2025).

This update does not introduce API changes, so the major version of gRPC remains unchanged.
