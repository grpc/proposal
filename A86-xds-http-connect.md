A86: xDS-Based HTTP CONNECT
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-10-25
* Discussion at: https://groups.google.com/g/grpc-io/c/WiqQ7h003fE

## Abstract

This document provides a design for gRPC to support configuring HTTP
CONNECT functionality via xDS.

## Background

gRPC currently supports HTTP CONNECT proxies, as described in [gRFC A1].
gRPC also supports obtaining configuration via xDS, as originally
described in [gRFC A27].  This document describes new functionality to
allow configuring use of an HTTP CONNECT proxy via xDS.

### Related Proposals: 
* [gRFC A1: HTTP CONNECT Proxy Support][gRFC A1]
* [gRFC A27: xDS-Based Global Load Balancing][gRFC A27]
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][gRFC A29]
* [gRFC A83: A83: xDS GCP Authentication Filter][gRFC A83]

[gRFC A1]: A1-http-connect-proxy-support.md
[gRFC A27]: A27-xds-global-load-balancing.md
[gRFC A29]: A29-xds-tls-security.md
[gRFC A83]: A83-xds-gcp-authn-filter.md

## Proposal

The xDS configuration for HTTP CONNECT is done via the
[`Http11ProxyUpstreamTransport`](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/extensions/transport_sockets/http_11_proxy/v3/upstream_http_11_connect.proto#L36)
transport socket wrapper.  This wrapper is sent in the CDS [`transport_socket`
field](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/cluster/v3/cluster.proto#L1099),
as a wrapper around whatever transport socket is actually used.

Note that the only transport socket implementation that gRPC currently
supports is `UpstreamTlsContext`, which is used to configure TLS
security, as per [gRFC A29].  We honor the xDS TLS configuration only if
the application uses `XdsCredentials` when creating the gRPC channel.
With this design, we will now honor the `Http11ProxyUpstreamTransport`
transport socket wrapper regardless of whether `XdsCredentials` is used.
The parsed form of the CDS resource will include a boolean field
indicating whether use of an HTTP proxy is enabled.

When validating the `Http11ProxyUpstreamTransport`, the nested
`transport_socket` field may be either unset or may contain an
`UpstreamTlsContext`, which we will validate exactly as if it had been
present in the `transport_socket` field in the CDS resource.  If there
is any other type in this field, we will NACK the resource.

We will use the nested `transport_socket` field the same way that we would
have if it had been the `transport_socket` field in the CDS resource.
The TLS configuration will be encoded in the parsed CDS resource in
exactly the same way, regardless of whether the `UpstreamTlsContext`
was present in the `Cluster.transport_socket` field or in the
`Http11ProxyUpstreamTransport.transport_socket` field.  The transport
security functionality will also remain unchanged: if `XdsCredentials`
is used and the TLS configuration is present, the TLS configuration will
be used; if `XdsCredentials` is used and the TLS configuration is absent,
then fallback credentials will be used; if `XdsCredentials` is not used,
then the TLS configuration will be ignored.

The `Http11ProxyUpstreamTransport` transport socket
wrapper looks for the proxy address in the EDS metadata,
both at the individual endpoint level (in the [`LbEndpoint.metadata`
field](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/endpoint/v3/endpoint_components.proto#L122))
and at the locality level (in the [`LocalityLbEndpoints.metadata`
field](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/endpoint/v3/endpoint_components.proto#L165)).

To support this, we will reuse the metadata registry mechanism described
for CDS in [gRFC A83] for the EDS metadata fields.  The parsed form
of the EDS resource will include a parsed metadata map at both the
individual endpoint and locality levels, using the same data type
previously introduced for CDS metadata.

We will support the [`envoy.config.core.v3.Address`
type](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/core/v3/address.proto#L175)
as a registered metadata type.  When validating this type, the
`socket_address` field must be set.  Inside of that field, the `address`
field must be set to an IPv4 or IPv6 address, and the `port_value` field
must be set.

The proxy address for a given endpoint will be set by looking for a
metadata entry of type `envoy.config.core.v3.Address` under the key
`envoy.http11_proxy_transport_socket.proxy_address`.  If that key is not
present, or if the key is present but the value has a different type,
the entry will be ignored.  The transport socket will look for the key
first in the endpoint-level metadata.  If not found there, it will look
in the locality-level metadata.  If not found in either place, then HTTP
CONNECT is not used (i.e., the behavior will be exactly the same as if the
`Http11ProxyUpstreamTransport` transport socket wrapper was not present).

If the `Http11ProxyUpstreamTransport` transport socket is used in CDS and
an endpoint has a proxy address, then the CDS LB policy must set some
appropriate resolver attributes on the endpoint to cause the specified
proxy to be used.  Note that this behavior may be triggered by a custom
proxy mapper (see [gRFC A1]).  The argument to the HTTP CONNECT request
sent on the wire will be the IP address and port of the endpoint.

### Temporary environment variable protection

This functionality will be guarded by the
`GRPC_EXPERIMENTAL_XDS_HTTP_CONNECT` environment variable.  This guard
will be removed when the feature passes interop tests.

## Rationale

Note that prior to this feature being enabled by default, gRPC will NACK a
CDS resource that specifies the `Http11ProxyUpstreamTransport` transport
socket.  Users will need to upgrade their clients before their control
plane starts sending CDS resources containing this transport socket.

## Implementation

C-core implementation: grpc/grpc#37800

Will also be implemented in Java and Go.
