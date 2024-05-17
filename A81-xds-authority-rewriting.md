A81: xDS Authority Rewriting
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-05-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC will add support for xDS-based authority rewriting.

## Background

gRPC supports getting routing configuration from an xDS server, as described
in gRFCs [A27] and [A28].  The xDS configuration can configure the
client to rewrite the authority header on requests, although gRPC does
not yet support this feature.

As mentioned in [gRFC A29][A29], there are use-cases for gRPC
that prohibit trusting the xDS server to control security-centric
configuration.  The authority rewriting feature falls under the same
umbrella as mTLS configuration.  As a result, if we add support for this
feature, we must do so in a way that allows disabling the feature for
these security-sensitive use-cases.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A28: xDS Traffic Splitting and Routing][A28]
* [gRFC A29: xDS mTLS Security][A29]
* [gRFC A30: xDS v3 Support][A30]
* [gRFC A31: xDS Timeout Support and Config Selector][A31]
* [gRFC A14: Channelz][A14]

[A27]: A27-xds-global-load-balancing.md
[A28]: A28-xds-traffic-splitting-and-routing.md
[A29]: A29-xds-tls-security.md
[A30]: A30-xds-v3.md
[A31]: A31-xds-timeout-support-and-config-selector.md
[A14]: A14-channelz.md

## Proposal

This proposal has several parts:
- Bootstrap config change: We will add a new server feature in the bootstrap
  config to indicate that we should allow authority rewriting.
- xDS resource validation: We will process authority rewriting fields in RDS
  resources and the hostname field in EDS resources, and return that
  information to the XdsClient watchers as part of the parsed resources.
- xDS ConfigSelector changes: We will change the xDS ConfigSelector to enable
  authority rewriting based on the chosen route.
- xds_cluster_impl LB policy changes: We will change the xds_cluster_impl LB
  policy to actually perform the authority rewriting was enabled by the
  ConfigSelector.

### Server Feature in Bootstrap Config

In order to address use-cases where authority rewriting may not be
acceptable from a security perspective, we will add a new server feature
to the bootstrap config.  The server feature will be specfied via the
`server_features` field described in [gRFC A30][A30].  The feature will
be the string `allow_authority_rewriting`.

### xDS Resource Validation

When validating an RDS resource, we will look at the
[`RouteAction.auto_host_rewrite`
field](https://github.com/envoyproxy/envoy/blob/b65de1f56850326e1c6b74aa72cb1c9777441065/api/envoy/config/route/v3/route_components.proto#L1173).
The boolean value of this field will be included in the parsed resource
struct that is passed to the XdsClient watcher.

When validting an EDS resource, we will look at the [`Endpoint.hostname`
field](https://github.com/envoyproxy/envoy/blob/b65de1f56850326e1c6b74aa72cb1c9777441065/api/envoy/config/endpoint/v3/endpoint_components.proto#L89).
The string value of this field will be included in a per-endpoint resolver
`hostname` attribute in the parsed resource struct that is passed to the
XdsClient watcher.  Note that the resolver attribute used here should
be a general-purpose one, not something specific to EDS; for example, in
the future, we may want to use this same attribute to expose per-endpoint
hostnames in channelz (see [gRFC A14][A14]).

For Logical DNS clusters, the same `hostname` resolver attribute will
be added to all endpoints.  It will be set to the name that is resolved
for the Logical DNS cluster.

TODO: do we need to support the [`RouteAction.append_x_forwarded_host`
field](https://github.com/envoyproxy/envoy/blob/b65de1f56850326e1c6b74aa72cb1c9777441065/api/envoy/config/route/v3/route_components.proto#L1222)?

### xDS ConfigSelector Changes

When the xDS ConfigSelector performs routing, if the chosen route has
`auto_host_rewrite` set to true, then the ConfigSelector will add a call
attribute to indicate that authority rewriting is enabled.  This call
attribute will be passed to the LB picker using the same mechanism
introduced in [A31] to pass the cluster name to the xds_cluster_manager
LB policy.

### xds_cluster_impl LB Policy Changes

The xds_cluster_impl policy will store the value of the `hostname`
attribute in the subchannel wrapper when its child policy creates a
subchannel.  In the xds_cluster_impl picker, if the `auto_host_rewrite`
call option is enabled and the `hostname` attribute on the subchannel
wrapper is non-empty, the picker will set the `:authority` header
to the value of the `hostname` attribute.

Note that we do not want to expose a general-purpose API for the LB
policy to be able to set the `:authority` header.  Instead, the
xds_cluster_impl picker will use an internal API for this.

### Temporary environment variable protection

Use of the RDS `auto_host_rewrite` field will be guarded by the
`GRPC_EXPERIMENTAL_XDS_AUTHORITY_REWRITE` env var.  The env var guard
will be removed once the feature passes interop tests.

## Rationale

We considered several alternatives to the server feature for enabling
use of authority rewriting:
- Using the existing XdsCredentials API (introduced in [gRFC A29][A29])
  to also enable authority rewriting.  This was deemed too complicated
  to implement in some languages, since we would have needed to provide
  a way to determine what channel credentials were used from the xDS
  routing code.
- Providing a channel option.  This would have required too much work on
  the part of applications to enable it on every individual channel.
  There were also some implementation challenges in some languages.  And
  it would not have provided a clean way to enforce disabling the
  setting in environments where that is required.
- Providing an environment variable.  This would also require
  applications to do work, although it would be in the deployment
  instead of in the application code.  And it would still not provide a
  clean way to enforce disabling the setting in environments where that
  is required.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
