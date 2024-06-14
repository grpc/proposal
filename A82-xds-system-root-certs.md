A82: xDS System Root Certificates
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-06-06
* Discussion at: https://groups.google.com/g/grpc-io/c/BgqeUU0q4fU

## Abstract

We will add a new xDS option to use the system's default root
certificates for TLS certificate validation.

## Background

Most service mesh workloads use mTLS, as described in [gRFC A29][A29].
However, there are cases where it is useful for applications to use
normal TLS rather than using certificates for workload identity, such as
when a mesh wants to move some workloads behind a reverse proxy.

gRPC already has code to find the system root certificates on various
platforms.  However, there is currently no way for the xDS control plane
to tell the client to use that functionality, at least not without the
cumbersome setup of duplicating that functionality in a certificate
provider config in the xDS bootstrap file.

### Related Proposals: 
* [gRFC A29: xDS mTLS Security][A29]

[A29]: A29-xds-tls-security.md

## Proposal

We have added a [`system_root_certs`
field](https://github.com/envoyproxy/envoy/blob/84d8fdd11e78013cd50596fa3b704e152512455e/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L399)
to the xDS `CertificateValidationContext` message (see envoyproxy/envoy#34235).
If this field is present and the `ca_certificate_provider_instance` field
is unset, system root certificates will be used for validation.

### xDS Resource Validation

When processing a CDS or LDS resource, we will look at this new field
if `ca_certificate_provider_instance` is unset.  The parsed CDS or LDS
resource delivered to the XdsClient watcher will indicate if system root
certs should be used.  If feasible, the parsed representation should be
structured such that it is not possible to indicate both a certificate
provider instance and using system root certs, since those options are
mutually exclusive.

### xds_cluster_impl LB Policy Changes

The xds_cluster_impl LB policy sets the configuration for the XdsCreds
functionality based on the CDS resource.  We will modify it such that if
the CDS resource indicates that system root certs are to be used, it
will configure XdsCreds to use system root certs.

### xDS-Enabled Server Changes

The xDS-enabled server code sets the configuration for XdsCreds
functionality based on the LDS resource.  We will modify it such that if
the LDS resource indicates that system root certs are to be used, it
will configure XdsCreds to use system root certs.

### XdsCredentials Changes

The XdsCredentials code will be modified such that if it is configured
to use system root certs, it will configure the TlsCreds code to do that.

### Temporary environment variable protection

Use of the `use_system_root_certs` field in CDS and LDS will be guarded
by the `GRPC_EXPERIMENTAL_XDS_SYSTEM_ROOT_CERTS` env var.  The env var
guard will be removed once the feature passes interop tests.

## Rationale

We already have code in gRPC to find the system root certs for various
platforms.  We don't want to have to reproduce that functionality in a
cert provider impl.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
