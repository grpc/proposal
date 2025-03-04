A87: gRPC OTel Metrics for xDS Outlier Detection
---
* Author(s): Pardhu Konakanchi, Sai Teja
* Approver:
* Status: Draft
* Implemented in: Core, Go
* Last updated: 03-04-25

## Abstract

This document proposes some new metrics that will be added in gRPC for xDS Client Outlier Detection

## Background

[A50: gRPC xDS Outlier Detection Support](https://github.com/grpc/proposal/blob/master/A50-xds-outlier-detection.md) is a spec for gRPC to support xDS Outlier Detection. The current implementation only offers debug and trace logging in terms of visibility, which can be insufficient for understanding and diagnosing decision making in large-scale production systems. 

Using A79: Non-per-call Metrics Architecture, it is possible to add granular metrics to make visibility into outlier detection easy for service owners utilizing gRPC.

### Related proposals: 
* [A50: gRPC xDS Outlier Detection Support](https://github.com/grpc/proposal/blob/master/A50-xds-outlier-detection.md)
* [A66: OpenTelemetry Metrics](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
* [A79: Non-per-call Metrics Architecture](https://github.com/grpc/proposal/blob/master/A79-non-per-call-metrics-architecture.md)

## Proposal

[A79’s](https://github.com/grpc/proposal/blob/master/A79-non-per-call-metrics-architecture.md) non-per-call metrics architecture fits perfectly into the metrics reporting solution. We propose to aim for parity where appropriate with [Envoy’s Outlier Detection metric collection](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats#outlier-detection-statistics) to be collected in gRPC

Outlier Detection metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | Indicates the target we are outlier detection on. (Same as the attribute defined in A66.) |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
|  grpc.xds_client.outlier_detection.ejections_enforced_total | Counter | {ejection} | 	grpc.target |	Total enforced ejections due to any outlier type |
|  grpc.xds_client.outlier_detection.ejections_active | Gauge |	{ejection} |	grpc.target |	Number of currently ejected hosts |
|  grpc.xds_client.outlier_detection.ejections_overflow |	Counter |	{ejection} |	grpc.target |	Number of ejections aborted due to max ejection percentage |
|  grpc.xds_client.outlier_detection.ejections_enforced_success_rate |	Counter |	{ejection} |	grpc.target |	Enforced success rate outlier ejections |
|  grpc.xds_client.outlier_detection.ejections_detected_success_rate |	Counter |	{ejection} |	grpc.target |	Detected (but unenforced) success rate outlier ejections |
|  grpc.xds_client.outlier_detection.ejections_enforced_failure_percentage |	Counter |	{ejection} |	grpc.target |	Enforced failure percentage outlier ejections |
|  grpc.xds_client.outlier_detection.ejections_detected_failure_percentage |	Counter |	{ejection} |	grpc.target |	Detected (but unenforced) failure percentage outlier ejections |

These metrics will all be available via global OpenTelemetry recorder. On any ejection/unejection, these metrics will be accordingly updated in the current outlier detection.

### Metric Stability

All metrics added in this proposal will start as experimental and therefore off by default. The long term goal will be to de-experimentalize them and have them be on by default, but the exact criteria for that change are TBD.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so it does not need environment variable protection.

## Rationale

The metrics defined here are generally a trade-off between the usefulness
of the metric and the cost of reporting it. As the design goal is offering parity to envoy metrics,
we decided to include any metric that was appropriate to gRPC outlier detection.

## Implementation

Dropbox is able to contribute towards a Core and Go implementation, in that order. Implementation of remaining gRPC languages is left for respective gRPC team or other contributors.
