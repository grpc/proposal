Title
----
* Author(s): Yash Tibrewal (yashkt@)
* Approver: Mark Roth (@markdroth)
* Status: Draft
* Implemented in: <language, ...>
* Last updated: Feb 22, 2024
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Describe a cross-language plugin architecture for collecting non-per-call metrics in the various gRPC implementations.

## Background

[A66: OpenTelemetry Metrics](A66-otel-stats.md) is a spec for gRPC to collect metrics for OpenTelemetry. It details an architecture that uses a CallTracer approach to record various per-call metrics like latency and message sizes. This approach works for per-call metrics but does not work so well for non-per-call metrics such as xDS metrics, load balancing policy metrics, resolver metrics and transport metrics which are not necessarily tied to a particular call. This calls for a newer architecture that allows to collect such metrics. This document proposes such an architecture.

### Related Proposals: 
* [A66: OpenTelemetry Metrics](A66-otel-stats.md)
* [A78: gRPC OTel Metrics for WRR, Pick First and XdsClient](https://github.com/grpc/proposal/pull/419)

## Proposal

### Overall Goals / Requirements
* Support non-per-call metrics - [A78: gRPC OTel Metrics for WRR, Pick First and XdsClient](https://github.com/grpc/proposal/pull/419) defines metrics for WRR, Pick-First and XdsClient components. In the future, we would want to record metrics for other components as well. Additionally, third-party plugins to gRPC (custom load balancing policies/resolvers) might want to use the same architecture to record metrics as well.
* Channel-scoped stats plugins - Applications should be able to register stats plugins such that metrics for only certain channels are recorded on those plugins. (Currently, implementations only support global registration of stats plugins.) There can be multiple stats plugins registered as well.
* gRPC/Third-party plugins control the metric/instrument definition - As gRPC library/component writers, we want to be in control of the metric definition that gRPC provides, irrespective of the stats plugin used. OpenTelemetry is the only supported stats plugin as of now, but that might change in the future. Third-party plugins to gRPC that choose to use this architecture would be responsible for the definitions of metrics exported by their plugins.
* Stats Plugins should be able to enable/disable metrics recording - gRPC has many components that would want to record metrics. Not all of those metrics will be interesting to stats plugins. Stats plugin writers should allow users to configure which metrics and enabled/disabled.

### Proposed Architecture

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
