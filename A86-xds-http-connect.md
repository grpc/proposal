A86: xDS-Based HTTP CONNECT
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-09-19
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
However, we will retain the original behavior for the underlying
transport socket wrapped by the `Http11ProxyUpstreamTransport` wrapper:
if the underlying transport socket is `UpstreamTlsContext`, then we will
honor it only if `XdsCredentials` is used, and if it is any other type,
we will not use it.

The `Http11ProxyUpstreamTransport` transport socket
wrapper looks for the proxy address in the EDS metadata,
both at the individual endpoint level (in the [`LbEndpoint.metadata`
field](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/endpoint/v3/endpoint_components.proto#L122))
and at the locality level (in the [`LocalityLbEndpoints.metadata`
field](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/endpoint/v3/endpoint_components.proto#L165)).

To support this, we will use the metadata mechanism described for CDS in
[gRFC A83] for the EDS metadata fields.  The parsed form of the EDS
resource will include a parsed metadata map at both the individual
endpoint and locality levels, using the same data type previously
introduced for CDS metadata.

We will support the [`envoy.config.core.v3.Address`
type](https://github.com/envoyproxy/envoy/blob/d6120f3c769e70c988ddcc5c7e9cbc2737b5f63c/api/envoy/config/core/v3/address.proto#L175)
as a registered metadata type.  When validating this type, the
`socket_address` field must be set.  Inside of that field, the `address`
field must be set to an IPv4 or IPv6 address, and the `port_value` field
must be set.

The `Http11ProxyUpstreamTransport` transport socket will look for a
metadata entry of type `envoy.config.core.v3.Address` under the key
`envoy.http11_proxy_transport_socket.proxy_address`.  If that key is not
present, or if the key is present but the value has a different type,
the entry will be ignored.  The transport socket will look for the key
first in the endpoint-level metadata.  If not found there, it will look
in the locality-level metadata.  If not found in either place, then HTTP
CONNECT is not used (i.e., the behavior will be exactly the same as if the
`Http11ProxyUpstreamTransport` transport socket wrapper was not present).

The argument to the HTTP CONNECT request sent on the wire will be the
endpoint's IP address that we are trying to connect to.

### Temporary environment variable protection

This functionality will be guarded by the
`GRPC_EXPERIMENTAL_XDS_HTTP_CONNECT` environment variable.  This guard
will be removed when the feature passes interop tests.

## Rationale

N/A

## Implementation

Will be implemented in C-core, Java, and Go.
