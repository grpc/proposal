Title
----
* Author(s): [Jim King, Emil Mukilic]
* Approver: Mark Roth, Nicolas Noble
* Status: In Review
* Implemented in: C++
* Last updated: 5-14-2018
* Discussion at:

## Abstract

The OpenCensus filter allows grpc to itegrate and collect stats and tracing data with OpenCensus (https://github.com/census-instrumentation/opencensus-cpp).
The Opencensus filter uses a number of internal APIs (src/core/lib/slice/slice_internal.h, src/cpp/common/channel_filter.h, src/core/lib/gprpp/orphanable.h,
src/core/lib/surface/call.h). Since these internal APIs are subject to change, in order to ease maintenance of the filter, we propose that the filter be moved to the
grpc repository (src/cpp/ext/filters/census/).

## Background

OpenCensus tracing and stats functionality is a desired feature for grpc C++ users. Integrating OpenCensus into grpc core as a direct dependency is not viable due
to OpenCensus dependencies which are not supported in grpc (abseil being a primary one).

### Related Proposals:
N/A

## Proposal

We propose that the OpenCensus filter (C++ grpc filter) which currently resides in the OpenCensus repository (https://github.com/census-instrumentation/opencensus-cpp)
be moved to the grpc repository (src/cpp/ext/filters/census/). The OpenCensus filter will be setup as an optional build target which users can include.  There is not
viable way to allow it to be included by default with grpc builds due to dependency conflicts.

## Rationale

The rationale behind this is primarily to ease maintenance of the filter as it depends on internal grpc APIs that are subject to change.  The API calls into the OpenCensus
library are unlikely to change in the future, so few if any API breaking changes are expected from the OpenCensus side.

## Implementation

The migration of the code is relatively simple.  The filter within OpenCensus (opencensus-cpp/opencensus/plugins/grpc/) will be migrated to grpc (src/cpp/ext/filters/census/).
A new build target will be introduced for the filter which will allow users to optionally include OpenCensus.

These changes are currently under review: https://github.com/grpc/grpc/pull/15070

## Open issues (if applicable)
N/A
