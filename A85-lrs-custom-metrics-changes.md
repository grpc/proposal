A85: Changes to xDS LRS Custom Metrics Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-07-19
* Discussion at: <google group thread> (filled after thread exists)

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

### Temporary environment variable protection

This option will be guarded by the
`GRPC_EXPERIMENTAL_XDS_ORCA_LRS_PROPAGATION` env var until it passes
interop tests.

Because this feature involves a change to gRPC's default behavior, we will
retain this env var for at least a couple of releases as a temporary
mechanism to get the old behavior back.  The default for the env var
will be set to true once it passes interop tests, but users will be able
to explicitly set it to false to get the old behavior.

## Rationale

N/A

## Implementation

Will be implemented in C-core, Java, and Go.
