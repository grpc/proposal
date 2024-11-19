A85: Changes to xDS LRS Custom Metrics Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-09-27
* Discussion at: https://groups.google.com/g/grpc-io/c/xawJAoE-pQE

## Abstract

We are adding xDS configuration to control which fields get propagated
from ORCA backend metric reports to LRS load reports.

## Background

In [gRFC A64][A64], we introduced support for propagating backend metrics
from ORCA load reports (see [gRFC A51][A51]) to LRS load reports.
That design says that we automatically propagate all entries in the
ORCA `named_metrics` map to LRS.  That approach has been found to be
insufficiently flexible, so we are introducing configuration to explicitly
control which fields are propagated.

### Related Proposals:
* [gRFC A51: Custom Backend Metrics Support][A51]
* [gRFC A64: xDS LRS Custom Metrics Support][A64]

[A51]: A51-custom-backend-metrics.md
[A64]: A64-lrs-custom-metrics.md

## Proposal

In envoyproxy/envoy#35210 and envoyproxy/envoy#35284, a new
`lrs_report_endpoint_metrics` field was added in CDS to control the set
of fields to propagate from ORCA to LRS.  In addition, new top-level
fields were added to LRS to efficiently support top-level metrics in ORCA.
We will honor all of this configuration in the gRPC client.

Note that this will change the gRPC client default behavior in two ways:

1. With this change, if the `lrs_report_endpoint_metrics` field is unset
   in the CDS resource, nothing will be propagated from ORCA to LRS.  To
   include the same metrics as before, the CDS resource must explicitly
   add `named_metrics.*` to `lrs_report_endpoint_metrics`.
2. If keys from the ORCA `named_metrics` map are propagated to LRS by
   the CDS `lrs_report_endpoint_metrics` field, the key names in LRS will
   be prepended with `named_metrics.` (e.g., if there is a key in the
   ORCA `named_metrics` map called `foo`, then it will appear in LRS as
   `named_metrics.foo`).

The specific changes needed in gRPC implementations are described in
the following subsections.

### CDS Resource Validation

The new `lrs_report_endpoint_metrics` field will be validated when
receiving a CDS resource, and the parsed form of this field will be
stored in the parsed form of the CDS resource that is provided to
XdsClient watchers.

To reduce memory usage, implementations may choose a more compact form
than the original set of strings.  For example, C-core will use the
following representation:

```c++
struct BackendMetricPropagation {
  static constexpr uint8_t kCpuUtilization = 1;
  static constexpr uint8_t kMemUtilization = 2;
  static constexpr uint8_t kApplicationUtilization = 4;
  static constexpr uint8_t kNamedMetricsAll = 8;

  // Bit mask of top-level fields to propagate.
  uint8_t propagation_bits = 0;

  // Keys from named_metrics map to propagate.
  // Used only if kNamedMetricsAll is not set in propagation_bits.
  absl::flat_hash_set<std::string> named_metric_keys;
};
```

For forward-compatibility, implementations should ignore any strings
that do not match a currently supported ORCA field.  Implementations
must support the following minimum set of fields:

- `cpu_utilization`
- `mem_utilization`
- `application_utilization`
- `named_metrics map`

### LRS APIs in XdsClient or LrsClient

The LRS API in XdsClient (or LrsClient, if the implementation has split
the LRS APIs into their own interface) currently provides a way to obtain
a locality stats handle that is keyed by LRS server, CDS and EDS
resource names, and locality.  We will add the propagation bits as
another dimension of this key.

When the xds_cluster_impl LB policy reports LRS metrics to the locality
stats handle at the end of an RPC, it will pass the ORCA data (if any).
Because the locality stats handle already has the propagation
configuration, it will know which ORCA fields to copy.

As an example, in C-core, we have the following APIs in LrsClient:

```c++
class ClusterLocalityStats {
 public:
  // ...

  // Reports metrics for a finished RPC.
  // backend_metrics points to the reported ORCA metrics, or null if none
  // were reported.
  void AddCallFinished(const BackendMetricData* backend_metrics,
                       bool fail = false);
};

// Adds locality stats for cluster_name and eds_service_name for the
// specified locality with the specified backend metric propagation.
RefCountedPtr<ClusterLocalityStats> AddClusterLocalityStats(
    std::shared_ptr<const XdsBootstrap::XdsServer> lrs_server,
    absl::string_view cluster_name, absl::string_view eds_service_name,
    RefCountedPtr<XdsLocalityName> locality,
    RefCountedPtr<const BackendMetricPropagation> backend_metric_propagation);
```

### Populating LRS Protos

When populating the LRS protos, if the propagation configuration says to
include any of the top-level fields (`cpu_utilization`,
`mem_utilization`, or `application_utilization`), then those will be
propagated to the new LRS fields of the same names in the
`UpstreamLocalityStats` message.

### Temporary environment variable protection

This option (both reading the new field in the CDS resource
and the new propagation behavior) will be guarded by the
`GRPC_EXPERIMENTAL_XDS_ORCA_LRS_PROPAGATION` env var until it passes
interop tests.

Because this feature involves a change to gRPC's default behavior, we will
retain this env var for at least a couple of releases as a temporary
mechanism to get the old behavior back.  The default for the env var
will be set to true once it passes interop tests, but users will be able
to explicitly set it to false to get the old behavior.

## Rationale

Storing the propagation info inside the locality stats handle from
the XdsClient or LrsClient is more efficient than the perhaps slightly
more obvious solution of handling this in the xds_cluster_impl policy
itself, since that would require storing the propagation information
both per-subchannel and per-call, which would increase memory usage.

## Implementation

C-core implementation in grpc/grpc#37467.

Will also be implemented in Java and Go.
