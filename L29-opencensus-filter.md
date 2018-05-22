C++ API Changes for OpenCensus Integration
----
* Author(s): Vizerai
* Approver: markdroth, vjpai
* Status: In Review
* Implemented in: C++
* Last updated: 2018-05-14
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/ft8UHY0EUz0

## Abstract

The OpenCensus filter allows grpc to integrate and collect stats and tracing data with OpenCensus (https://github.com/census-instrumentation/opencensus-cpp). The OpenCensus filter uses a number of internal APIs (src/core/lib/slice/slice_internal.h, src/cpp/common/channel_filter.h, src/core/lib/gprpp/orphanable.h, src/core/lib/surface/call.h). Since these internal APIs are subject to change, in order to ease maintenance of the filter, we propose that the filter be moved to the grpc repository (src/cpp/ext/filters/census/). This will introduce an API change for grpc++ when OpenCensus is used, which will require the user to register the OpenCensus filter with grpc.  This is in addition to any views and exporters that would need to be registered to use OpenCensus.

## Background

OpenCensus tracing and stats functionality is a desired feature for grpc C++ users. Integrating OpenCensus into grpc core as a direct dependency is not viable due to OpenCensus dependencies which are not supported in grpc (abseil being a primary one).

### Related Proposals:
N/A

## Proposal

We propose that the OpenCensus filter (C++ grpc filter) which currently resides in the OpenCensus repository (https://github.com/census-instrumentation/opencensus-cpp) be moved to the grpc repository (src/cpp/ext/filters/census/). The OpenCensus filter will be setup as an optional build target which users can include.  There no viable way to include it by default with the default grpc build due to dependency conflicts. Users will have to manually enable the filter by using a filter registration call.

The major changes introduced by this are as follows:
  1) A new optional dependency on OpenCensus for bazel builds. This will be a new build target in the Bazel build file. Currently only the Bazel build will be supported.  There will be no git submodule for OpenCensus, and no make/cmake support (this may be added at a later time).
  2) Two new public API calls will be introduced to enable the OpenCensus filter for tracing and stats collection and register default views. The registration call must be made to register the plugin mechanism and initialize OpenCensus. The call to register default views is optional and does not need to be used if the user will be registering their own views.

  * The public API calls will reside in include/grpcpp/opencensus.h :

        void RegisterOpenCensusPlugin();
        void RegisterGrpcViewsForExport();

## Rationale

The rationale behind moving the filter code is primarily to ease maintenance of the filter as it depends on internal grpc APIs that are subject to change.  The API calls into the OpenCensus library are unlikely to change in the future, so few if any API breaking changes are expected from the OpenCensus side. An initialization call is required, because there is no viable way to have it enabled by default. Building it directly with grpc is not possible as it introduces dependency conflicts. Dynamic initialization from linking through weak symbols is not available on all platforms (namely Windows).

## Implementation

The migration of the code is relatively simple.  The filter within OpenCensus (opencensus-cpp/opencensus/plugins/grpc/) will be migrated to grpc (src/cpp/ext/filters/census/). A new build target will be introduced for the filter which will allow users to optionally include OpenCensus. This target contains the registration function needed to initialize the OpenCensus filter.

These changes are currently under review: https://github.com/grpc/grpc/pull/15070

## Open issues (if applicable)
N/A
