A91: gRPC Metrics for Outlier Detection
---
* Author(s): @pardhukonakanchi, @huntsman90
* Approver: @markdroth
* Status: In Review
* Implemented in:
* Last updated: 2025-03-05
* Discussion at: https://groups.google.com/g/grpc-io/c/iMezDbq5U-g

## Abstract

This document proposes some new metrics that will be added in gRPC for xDS Client Outlier Detection.

## Background

[A50: gRPC xDS Outlier Detection Support][A50] is a spec for gRPC to support Outlier Detection. The current implementation only offers debug and trace logging in terms of visibility, which can be insufficient for understanding and diagnosing decision making in large-scale production systems. 

Using [A79: Non-per-call Metrics Architecture][A79], it is possible to add granular metrics to make visibility into outlier detection easy for service owners utilizing gRPC.

### Related proposals: 
* [A50: gRPC xDS Outlier Detection Support][A50]
* [A66: OpenTelemetry Metrics][A66]
* [A79: Non-per-call Metrics Architecture][A79]
* [A75: xDS Aggregate Cluster Behavior Fixes][A75]
* [A89: Backend Service Metric Label][A89]

[A50]: A50-xds-outlier-detection.md
[A66]: A66-otel-stats.md
[A75]: A75-xds-aggregate-cluster-behavior-fixes.md
[A79]: A79-non-per-call-metrics-architecture.md
[A89]: A89-backend-service-metric-label.md

## Proposal

[A79]’s non-per-call metrics architecture fits perfectly into the metrics reporting solution. We propose to aim for parity where appropriate with [Envoy’s Outlier Detection metric collection](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats#outlier-detection-statistics) to be collected in gRPC.

Outlier Detection metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | Indicates the target we are running outlier detection on, as described in [A66]. |
| grpc.lb.outlier_detection.detection_method | required | Indicates the method with which we detected outlier. Currently one of {"success_rate", "failure_percentage"}
| grpc.lb.outlier_detection.unenforced_reason | required | Indicates the reason we did not eject a detected outlier. Currently one of {"enforcement_percentage", "max_ejection_overflow"}
| grpc.lb.backend_service | optional | The backend service to which the traffic is being sent, as described in [A89]. Note that this label will be supported only if [A75] has already been implemented |

The `grpc.lb.backend_service` label will be populated based on the resolver attribute passed down from the cds policy, as described in A89.

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
|  grpc.lb.outlier_detection.ejections_enforced |	Counter |	{ejection} |	grpc.target, grpc.lb.backend_service, grpc.lb.outlier_detection.detection_method  |	Enforced outlier ejections by ejection reason |
|  grpc.lb.outlier_detection.ejections_unenforced |	Counter |	{ejection} |	grpc.target, grpc.lb.backend_service, grpc.lb.outlier_detection.detection_method, grpc.lb.outlier_detection.unenforced_reason |	Unenforced outlier ejections due to either max ejection percentage or enforcement_percentage |

On any detection and ejection/unejection, these metrics will be accordingly updated.

### Metric Stability

All metrics added in this proposal will start as experimental and therefore off by default. The long term goal will be to de-experimentalize them and have them be on by default, but the exact criteria for that change are TBD.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so it does not need environment variable protection.

## Rationale

The metrics defined here are generally a trade-off between the usefulness
of the metric and the cost of reporting it. As the design goal is offering parity to envoy metrics,
we decided to include any metric that was appropriate to gRPC outlier detection.

One change from envoy metrics was instead of reporting all detected ejections (enforced or unenforced) for each algorithm type as its own metric, we opted to simply report enforced and unenforced ejections separately. This reduces cost of any detected outlier by 1 metric in the enforced case, and the unenforced count is more likely the direct information a user of the "detected" metric in Envoy is seeking.

Additionally, we combined the ejections_enforced and ejections_unenforced into one metric with a label to provide the ejection/unejection reason.

## Implementation

Dropbox is able to contribute towards a Core and Go implementation, in that order. Implementation of remaining gRPC languages is left for respective gRPC team or other contributors.
