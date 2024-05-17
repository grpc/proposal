A82: xDS System Root Certificates
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-05-17
* Discussion at: <google group thread> (filled after thread exists)

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

We will add boolean a field to the xDS `CertificateValidationContext` message
called `use_system_root_certs`.  If this field is true and the
`ca_certificate_provider_instance` field is unset, system root
certificates will be used for validation.

### xDS Resource Validation

When processing a CDS or LDS resource, we will look at this new field if
`ca_certificate_provider_instance` is unset.  The value of the field
will be specified in the parsed CDS or LDS resource delivered to the
XdsClient watcher.

### xds_cluster_impl LB Policy Changes

The xds_cluster_impl LB policy sets the configuration for the XdsCreds
functionality based on the CDS resource.  We will modify it such that if
the CDS resource does not have the `ca_certificate_provider_instance`
set, it will instead look at `use_system_root_certs`.  In that case, it
will configure XdsCreds to use system root certs.

### xDS-Enabled Server Changes

The xDS-enabled server code sets the configuration for XdsCreds
functionality based on the LDS resource.  We will modify it such that if
the LDS resource does not have the `ca_certificate_provider_instance`
set, it will instead look at `use_system_root_certs`.  In that case, it
will configure XdsCreds to use system root certs.

TODO: Should we support this option on the server side?  Maybe we want
it only on the client side, since that's the use-case we're concerned
about here.

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
