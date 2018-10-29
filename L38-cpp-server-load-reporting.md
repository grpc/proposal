C++ API changes for Server Load Reporting
----
* Author(s): AspirinSJL
* Approver: markdroth, vjpai
* Status: Draft
* Implemented in: C++
* Last updated: 2018-07-16
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/6p8tL8UdG9s

## Abstract

As part of the load balancing functionality, the server-side load reporting service is to be introduced. A `grpc::load_reporter::LoadReportingServiceServerBuilderOption` will be used to enable that service. A utility function `grpc::load_reporter::AddLoadReportingCost()` can be used to report the customized call metrics.

## Background

A load balancer needs to get the load data of the servers in order to balance the load among the servers accordingly. One possible approach to get the load data is to request the load data from the servers via RPCs. That requires the servers to register an additional service to report the load data.


### Related Proposals: 
N/A

## Proposal

We propose a new public header `include/grpcpp/ext/server_load_reporting.h` to use the server load reporting service. Note that this public header (and the load reporting service) is only available when the binary is built with the `grpcpp_server_load_reporting` library. That library is only available in Bazel build because it depends on OpenCensus which can only be built with Bazel.

1. The header contains a `grpc::load_reporter::LoadReportingServiceServerBuilderOption`. The user should set that option in the `grpc::ServerBuilder` to enable the load reporting service on a server.
2. The header also contains a utility function `void grpc::load_reporter::AddLoadReportingCost(grpc::ServerContext* ctx, const grpc::string& cost_name, double cost_value);`. Besides the canonical call metrics (e.g., the number of calls started, the total bytes received, etc), the user can use this function to add other customized call metrics from their own normal services.

We also propose a cleanup change.

- The previous public header `include/grpc/load_reporting.h` will be renamed to `include/grpc/server_load_reporting.h` to distinguish from the client load reporting in grpclb.

## Rationale

1. The load reporting service is opted in via a `grpc::ServerBuilderOption` instead of being automatically enabled because this service should only be enabled when there will be a load balancer requesting the load data. Otherwise, the load data will be accumulated for nothing.
2. The header under `include/grpc` is renamed to have a "server_" prefix because we actually have [client load reporting in grpclb](https://github.com/grpc/grpc/blob/85daf2db65d60ebd63936a936d69c63777123d10/src/core/ext/filters/client_channel/lb_policy/grpclb/client_load_reporting_filter.h). The new name will also be more consistent with `include/grpcpp/ext/server_load_reporting.h`.

## Implementation

Implementation is completed. The new APIs are currently under `experimental` namespace.

- The public APIs are in `include/grpcpp/ext/server_load_reporting.h`.
- The protobuf definition is in `src/proto/grpc/lb/v1/load_reporter.proto`.
- The implementation is under `src/cpp/server/load_reporter`.
