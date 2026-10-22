## L123: Deprecate and Remove C++ OpenCensus and dependent APIs

*   Author(s): Yash Tibrewal (@yashykt)
*   Approver: Mark Roth (@markdroth) , Esun Kim (@veblush)
*   Status: In Review
*   Implemented in: C++
*   Last updated: May 13, 2025
*   Discussion at: https://groups.google.com/g/grpc-io/c/ASVIZhcZgl0

## Abstract

Remove gRPC C++ OpenCensus and dependent APIs.

## Background

OpenCensus was
[sunsetted in May, 2023](https://opentelemetry.io/blog/2023/sunsetting-opencensus/),
and the
[C++ OpenCensus repository](https://github.com/census-instrumentation/opencensus-cpp)
has been archived (no longer maintained).

Furthermore, the last available commit of OpenCensus C++ does not build with the
[latest candidate release of the Abseil library](https://github.com/abseil/abseil-cpp/releases/tag/20250512.rc1)
(OpenCensus C++ relied on internal headers and methods from Abseil.)

## Proposal

Deprecate the
[C++ OpenCensus](https://github.com/grpc/grpc/blob/v1.72.x/include/grpcpp/opencensus.h)
and dependent
[gRPC C++ GCP Observability](https://github.com/grpc/grpc/blob/master/include/grpcpp/ext/gcp_observability.h)
APIs and remove them in a subsequent release. Users still utilizing OpenCensus
APIs should migrate to OpenTelemetry.

## Rationale

Given gRPC C++'s dependency on Abseil, maintaining OpenCensus support in future
versions is not feasible. Deleting these APIs will allow gRPC C++ users to
upgrade the version of the Abseil library used.

## Implementation

https://github.com/grpc/grpc/pull/39554 marks these APIs as deprecated. A
subsequent release will delete these APIs.
