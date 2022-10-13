L104: C++ OpenCensus Plugin Public APIs
----
* Author(s): yashykt
* Approver: markdroth
* Status: Draft
* Implemented in:
* Last updated: Oct 12, 2022
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Move the gRPC C++ OpenCensus plugin APIs that were designed as public API to include/grpcpp/opencensus.h from src/cpp/ext/cpp/filter/census.

## Background

The gRPC C++ OpenCensus filter was proposed in [L29-cpp-opencensus-filter](L29-cpp-opencensus-filter.md) and added two public APIs for registering the OpenCensus plugin and registering the default views. It seems that it was intended for users to be able to create their own views or some of the additional views provided in `src/cpp/ext/cpp/filter/census/grpc_plugin.h`, but those APIs were not moved to `include/grpcpp/opencensus.h`. This is also the case for the class `grpc::CensusContext` currently available in `src/cpp/ext/cpp/filter/census/context.h` which users should be able to use in conjunction with the `set_cencus_context` API on `grpc::ClientContext`.


### Related Proposals: 
* [L29-cpp-opencensus-filter](L29-cpp-opencensus-filter.md)

## Proposal

From `src/cpp/ext/cpp/filter/census/grpc_plugin.h`, move the following APIs to `include/grpcpp/opencensus.h` -
```
// The tag keys set when recording RPC stats.
::opencensus::tags::TagKey ClientMethodTagKey();
::opencensus::tags::TagKey ClientStatusTagKey();
::opencensus::tags::TagKey ServerMethodTagKey();
::opencensus::tags::TagKey ServerStatusTagKey();

// Names of measures used by the plugin--users can create views on these
// measures but should not record data for them.
extern const absl::string_view kRpcClientSentMessagesPerRpcMeasureName;
extern const absl::string_view kRpcClientSentBytesPerRpcMeasureName;
extern const absl::string_view kRpcClientReceivedMessagesPerRpcMeasureName;
extern const absl::string_view kRpcClientReceivedBytesPerRpcMeasureName;
extern const absl::string_view kRpcClientRoundtripLatencyMeasureName;
extern const absl::string_view kRpcClientServerLatencyMeasureName;
extern const absl::string_view kRpcClientStartedRpcsMeasureName;
extern const absl::string_view kRpcClientRetriesPerCallMeasureName;
extern const absl::string_view kRpcClientTransparentRetriesPerCallMeasureName;
extern const absl::string_view kRpcClientRetryDelayPerCallMeasureName;

extern const absl::string_view kRpcServerSentMessagesPerRpcMeasureName;
extern const absl::string_view kRpcServerSentBytesPerRpcMeasureName;
extern const absl::string_view kRpcServerReceivedMessagesPerRpcMeasureName;
extern const absl::string_view kRpcServerReceivedBytesPerRpcMeasureName;
extern const absl::string_view kRpcServerServerLatencyMeasureName;
extern const absl::string_view kRpcServerStartedRpcsMeasureName;

// Canonical gRPC view definitions.
const ::opencensus::stats::ViewDescriptor& ClientSentMessagesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor& ClientSentBytesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor&
ClientReceivedMessagesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor&
ClientReceivedBytesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor& ClientRoundtripLatencyCumulative();
const ::opencensus::stats::ViewDescriptor& ClientServerLatencyCumulative();
const ::opencensus::stats::ViewDescriptor& ClientStartedRpcsCumulative();
const ::opencensus::stats::ViewDescriptor& ClientCompletedRpcsCumulative();
const ::opencensus::stats::ViewDescriptor& ClientRetriesPerCallCumulative();
const ::opencensus::stats::ViewDescriptor& ClientRetriesCumulative();
const ::opencensus::stats::ViewDescriptor&
ClientTransparentRetriesPerCallCumulative();
const ::opencensus::stats::ViewDescriptor& ClientTransparentRetriesCumulative();
const ::opencensus::stats::ViewDescriptor& ClientRetryDelayPerCallCumulative();

const ::opencensus::stats::ViewDescriptor& ServerSentBytesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor&
ServerReceivedBytesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor& ServerServerLatencyCumulative();
const ::opencensus::stats::ViewDescriptor& ServerStartedCountCumulative();
const ::opencensus::stats::ViewDescriptor& ServerStartedRpcsCumulative();
const ::opencensus::stats::ViewDescriptor& ServerCompletedRpcsCumulative();
const ::opencensus::stats::ViewDescriptor& ServerSentMessagesPerRpcCumulative();
const ::opencensus::stats::ViewDescriptor&
ServerReceivedMessagesPerRpcCumulative();

const ::opencensus::stats::ViewDescriptor& ClientSentMessagesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ClientSentBytesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ClientReceivedMessagesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ClientReceivedBytesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ClientRoundtripLatencyMinute();
const ::opencensus::stats::ViewDescriptor& ClientServerLatencyMinute();
const ::opencensus::stats::ViewDescriptor& ClientStartedRpcsMinute();
const ::opencensus::stats::ViewDescriptor& ClientCompletedRpcsMinute();
const ::opencensus::stats::ViewDescriptor& ClientRetriesPerCallMinute();
const ::opencensus::stats::ViewDescriptor& ClientRetriesMinute();
const ::opencensus::stats::ViewDescriptor&
ClientTransparentRetriesPerCallMinute();
const ::opencensus::stats::ViewDescriptor& ClientTransparentRetriesMinute();
const ::opencensus::stats::ViewDescriptor& ClientRetryDelayPerCallMinute();

const ::opencensus::stats::ViewDescriptor& ServerSentMessagesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ServerSentBytesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ServerReceivedMessagesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ServerReceivedBytesPerRpcMinute();
const ::opencensus::stats::ViewDescriptor& ServerServerLatencyMinute();
const ::opencensus::stats::ViewDescriptor& ServerStartedRpcsMinute();
const ::opencensus::stats::ViewDescriptor& ServerCompletedRpcsMinute();

const ::opencensus::stats::ViewDescriptor& ClientSentMessagesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ClientSentBytesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ClientReceivedMessagesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ClientReceivedBytesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ClientRoundtripLatencyHour();
const ::opencensus::stats::ViewDescriptor& ClientServerLatencyHour();
const ::opencensus::stats::ViewDescriptor& ClientStartedRpcsHour();
const ::opencensus::stats::ViewDescriptor& ClientCompletedRpcsHour();
const ::opencensus::stats::ViewDescriptor& ClientRetriesPerCallHour();
const ::opencensus::stats::ViewDescriptor& ClientRetriesHour();
const ::opencensus::stats::ViewDescriptor&
ClientTransparentRetriesPerCallHour();
const ::opencensus::stats::ViewDescriptor& ClientTransparentRetriesHour();
const ::opencensus::stats::ViewDescriptor& ClientRetryDelayPerCallHour();

const ::opencensus::stats::ViewDescriptor& ServerSentMessagesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ServerSentBytesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ServerReceivedMessagesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ServerReceivedBytesPerRpcHour();
const ::opencensus::stats::ViewDescriptor& ServerServerLatencyHour();
const ::opencensus::stats::ViewDescriptor& ServerStartedCountHour();
const ::opencensus::stats::ViewDescriptor& ServerStartedRpcsHour();
const ::opencensus::stats::ViewDescriptor& ServerCompletedRpcsHour();
```

From `src/cpp/ext/filter/census/context.h`, move the class grpc::CensusContext to `include/grpcpp/opencensus.h`.


## Rationale

Public APIs should be available in the `include/` directory. Headers in `src/` are not considered public API and are not stable.


## Implementation

Implementation in https://github.com/grpc/grpc/pull/31341.
