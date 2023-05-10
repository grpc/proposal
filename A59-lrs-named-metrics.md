A59: xDS LRS Named Metrics Support
----
* Author: yousukseung
* Approver(s): markdroth
* Status: Ready for Implementation
* Implemented in: C-core
* Last updated: 2023-05-10
* Discussion at: TBA

## Abstract

This proposal describes how the gRPC xDS client will support named metrics from the backends in load reports to the LRS server at the locality level.

## Background

The gRPC xDS client currently includes request counts in LRS load reports. These are locally counted simple request counts, however there are cases where the service needs to include custom data from the backends in LRS load reports for load balancing purposes. gRPC now supports ORCA based load reporting from the backend and this proposal describes how to integrate it with LRS load reporting in the gRPC xDS client.

### Related Proposals: 
[A51: Custom Backend Metrics Support][A51]
[Open Request Cost Aggregation (ORCA)][ORCA]

## Proposal

### ORCA Integration

The gRPC xDS client will include the entire `named_metrics` field in ORCA load reports from the backend. They will be considered application specific opaque values with no validation.

```textproto
message OrcaLoadReport {
  // Application specific opaque metrics.
  map<string, double> named_metrics = 8;
}
```

### Per-Request Support Only

Named metrics in LRS load reports will only include `named_metrics` from per-request ORCA load reports and not OOB load reports. See [gRFC A51][A51] for more on per-request and OOB load reports.

### Aggregation

Each per-request ORCA load report will be associated with one request for the aggregation purpose regardless of whether `named_metrics` has entries or not, or whether a certain key exists or not in it. All values in `named_metrics` will be considered cumulative and tracked at the locality level.

For example, following ORCA load reports
```textproto
// report 1/3
rps_fractional: 10.0
named_metrics { key: "key1" value: 1.0 }
named_metrics { key: "key2" value: 2.0 }
// report 2/3
rps_fractional: 20.0
named_metrics { key: "key2" value: 3.0 }
named_metrics { key: "key3" value: 4.0 }
// report 3/3
rps_fractional: 30.0
```
will be aggreated to as follows.
|field|value|
|------|---|
|total requests|3|
|`key1`|1.0|
|`key2`|5.0|
|`key3`|4.0|

### LRS Load Report

The gRPC xDS client will include named metrics in the `load_metric_stats` field in the locality stats. Each LRS report will include aggregated named metrics with keys reported since the last report. Locally aggregated stats will be cleared and the associated total request count will be reset to zero after each LRS load report is generated.

An example LRS load report with named metrics is as follows.
```textproto
// LRS report (envoy.service.load_stats.v3.LoadStatsRequest)
cluster_stats {
  // …
  upstream_locality_stats {
     total_successful_requests: 10
     total_requests_in_progress: 2
     total_error_requests: 4
     total_issued_requests = 13
     load_metric_stats {
        metric_name: "key1"
        num_requests_finished_with_metric: 10
        total_metric_value: 12.3
     } 
     load_metric_stats {
        metric_name: "key2"
        num_requests_finished_with_metric: 10
        total_metric_value: 23.4
     }
  }
}
```
## Implementation

This is implemented in C-core with [#32690][PR_32690], and will be implemented in Java and Go.

[A51]: https://github.com/grpc/proposal/blob/master/A51-custom-backend-metrics.md
[ORCA]: https://github.com/envoyproxy/envoy/issues/6614
[PR_32690]: https://github.com/grpc/grpc/pull/32690
