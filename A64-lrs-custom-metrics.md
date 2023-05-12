A64: xDS LRS Custom Metrics Support
----
* Author: yousukseung
* Approver(s): markdroth
* Status: Implemented
* Implemented in: C-core
* Last updated: 2023-05-10
* Discussion at: https://groups.google.com/g/grpc-io/c/Cs7ffkO1wUA

## Abstract

This proposal describes how the gRPC xDS client will support custom metrics from the backends in load reports to the LRS server at the locality level.

## Background

The initial xDS functionality in the gRPC client described in [gRFC A27][A27] includes the ability to report load to the control plane via LRS. However, those load reports currently contain only basic request counts tracked by the client. [gRFC A51][A51] added support for backends to report custom backend metrics to the client via the [ORCA protocol][ORCA]. This proposal adds the ability for the client to propagate those backend metrics to the control plane in the LRS load reports.

### Related Proposals:
* [A27: xDS-Based Global Load Balancing][A27]
* [A51: Custom Backend Metrics Support][A51]

## Proposal

### ORCA Integration

The gRPC xDS client will include the entire `named_metrics` field in ORCA load reports from the backend. They will be considered application specific opaque values with no validation.

### Per-Request Support Only

Custom metrics in LRS load reports will only include `named_metrics` from per-request ORCA load reports and not OOB load reports. See [gRFC A51][A51] for more on per-request and OOB load reports.

### Aggregation

Each per-request ORCA load report will be associated with one request for the aggregation purpose. The value and count of each entry in `named_metrics` will be aggregated separately. All values will be considered cumulative and will be aggregated using addition. They will be tracked at the locality level.

For example, following ORCA load reports
```textproto
// report 1/3
named_metrics { key: "key1" value: 1.0 }
named_metrics { key: "key2" value: 2.0 }
// report 2/3
named_metrics { key: "key2" value: 3.0 }
named_metrics { key: "key3" value: 4.0 }
// report 3/3
// (no named_metrics)
```
will be aggreated to as follows.
|field|value|number of requests|
|------|---|---|
|`key1`|1.0|1|
|`key2`|5.0|2|
|`key3`|4.0|1|

### LRS Load Report

The gRPC xDS client will include custom metrics in the `load_metric_stats` field in the locality stats. Each LRS report will include aggregated custom metrics with keys reported since the last report. Locally aggregated stats will be cleared and the associated total request counts will be reset to zero after each LRS load report is generated.

An example LRS load report with custom metrics is as follows.
```textproto
// LRS report (envoy.service.load_stats.v3.LoadStatsRequest)
cluster_stats {
  // …
  upstream_locality_stats {
     load_metric_stats {
        metric_name: "key1"
        num_requests_finished_with_metric: 1
        total_metric_value: 1.0
     }
     load_metric_stats {
        metric_name: "key2"
        num_requests_finished_with_metric: 2
        total_metric_value: 5.0
     }
     load_metric_stats {
        metric_name: "key3"
        num_requests_finished_with_metric: 3
        total_metric_value: 4.0
     }
  }
}
```
### xDS Integration

The `xds_cluster_impl` LB policy, which is already tracking call status for the client-tracked request counts, will be changed to also get the per-request backend metric data reported by the backend. It will report that data to the `XdsClient` along with the request counts. The `XdsClient` will perform aggregation of the data and include it in LRS load reports.

## Implementation

This is implemented in C-core with [#32690][PR_32690], and will be implemented in Java and Go.

[A27]: https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md
[A51]: https://github.com/grpc/proposal/blob/master/A51-custom-backend-metrics.md
[ORCA]: https://github.com/envoyproxy/envoy/issues/6614
[PR_32690]: https://github.com/grpc/grpc/pull/32690
