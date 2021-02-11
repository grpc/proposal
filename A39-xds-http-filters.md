A39: xDS HTTP Filter Support
----
* Author(s): Mark D. Roth (markdroth)
* Approver: ejona86, dfawley
* Status: In Review
* Implemented in: 
* Last updated: 2021-02-10
* Discussion at: https://groups.google.com/g/grpc-io/c/M-l8k2v5snY

## Abstract

The purpose of this proposal is to describe how gRPC will support HTTP
filters configured via the xDS API.

## Background

gRPC currently supports obtaining configuration via the
[xDS API](https://github.com/envoyproxy/data-plane-api), as per
[gRFC A27](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

This proposal details how gRPC will add support for xDS functionality that is
configured via [HTTP
filters](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters).  Initially, we will not provide any sort of public
API that third parties can use to implement their own filters, although
that may come later.  For now, we are focusing on creating a structure
that will enable us to implement functionality that, in Envoy, is
provided by its out-of-the-box filters (e.g., fault injection or
authorization), which we also want to be provided out-of-the-box in gRPC.

### Related Proposals: 

- [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
- [A30: xDS v3 Support](https://github.com/grpc/proposal/blob/master/A30-xds-v3.md)
- [A31: gRPC xDS Timeout Support and Config Selector Design](https://github.com/grpc/proposal/blob/master/A31-xds-timeout-support-and-config-selector.md)
- [A33: xDS Client-Side Fault Injection](https://github.com/grpc/proposal/pull/201) (pending)
- [A36: xDS-Enabled Servers](https://github.com/grpc/proposal/pull/214) (pending)

## Proposal

In the gRPC client channel, the xDS HTTP filters will run in between
name resolution and load balancing.  The set of filters to run will be
determined by the `ConfigSelector` returned by the resolver, which is
where gRPC implements the xDS routing functionality.

In C-core, the xDS HTTP filters will be implemented as C-core filters.
In Java and Go, they will be implemented as interceptors.

We will also support xDS HTTP filters in the gRPC server.  However,
unlike Envoy, gRPC cannot use the same filter implementations on the
client and server side, because the operations are inverse (e.g., a
fault-injection filter on the client side needs to take action on send
operations, whereas the same filter on the server side needs to take
action on receive operations).  Therefore, gRPC will effectively have
independent sets of supported filters on the client and server side
(i.e., just because a filter is supported on the client side does not
mean that it will also be accepted on the server side, and vice-versa).

### Limitations

gRPC will support HTTP filters in xDS v3 only.  Filters will continue to
be ignored when speaking xDS v2.  This will avoid the need to support
both v2 and v3 type names in filter config message types.

In Envoy, HTTP filters have access to an API to tell Envoy to recompute
the route, which they can use after modifying request state that can
affect the choice of route (e.g., changing a header).  Initially, gRPC will
not support any such mechanism, although this is something that we may
add later, if/when it becomes necessary.

### xDS API Fields

gRPC will add support for the following fields in the xDS APIs:
- [`HttpConnectionManager.http_filters`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L273)
- [`VirtualHost.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/config/route/v3/route_components.proto#L142)
- [`Route.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/config/route/v3/route_components.proto#L252)
- [`WeightedCluster.ClusterWeight.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/config/route/v3/route_components.proto#L366)

The `HttpConnectionManager.http_filters` field is an ordered list of
filters.  In each entry, gRPC will look at the following fields:
- [`name`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L833):
  A logical name for the filter instance.  This name will be used when
  applying overrides from the virtual host, route, or weighted cluster entry.
  The logical instance name also allows configuring multiple instances of the
  same filter, which can be useful for cases like authorization policies.
- [`typed_config`](https://github.com/envoyproxy/envoy/blob/2ee9543fbb0fccdb69d4dcd8cb57a19743afb94b/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L838):
  This field contains the configuration message for the filter.  gRPC
  will contain a registry of filter implementations, keyed by the proto
  message type used for its configuration; thus, when gRPC sees an entry
  with a given proto message type, it will know which filter to
  instantiate.  Note that as a special case, if the proto message type is
  `udpa.type.v1.TypedStruct`, then gRPC will look inside that type's
  [`type_url`](https://github.com/cncf/udpa/blob/cc1b757b3eddccaaaf0743cbb107742bb7e3ee4f/udpa/type/v1/typed_struct.proto#L38)
  field to determine the actual type; this allows for third-party
  plugins to encode their configurations as a `google.protobuf.Struct`
  proto instead of using a concrete proto message type that the client
  may not be able to handle.

When gRPC receives the `HttpConnectionManager` proto in the LDS
response, it will validate the list of filters as follows:
- Every filter config in the list must be of a known type, as identified
  by the `type_url` field.  Any unknown config type will cause the Listener
  resource to be NACKed.  Note that the set of known filter config types
  will be different on the gRPC client and server; if the
  `HttpConnectionManager` proto is inside an HTTP API Listener, it will
  use the client set, whereas if it is inside a TCP Listener, it will
  use the server set.
- The fields in the filter config should be validated by the filter
  implementation.  Any validation error will cause the Listener resource
  to be NACKed.
- Every filter in the list must have a unique instance name.  Any
  duplicate instance names will cause the Listener resource to be NACKed.

Note that gRPC will support configuring multiple instances of the same
filter implementation, each with its own name and config.

In each of the `VirtualHost`, `Route`, and `WeightedCluster.ClusterWeight`
protos, there is a field called `typed_per_filter_config`, which is a map
from filter instance name to a corresponding filter-specific configuration
proto message.  (Note that Envoy currently keys these maps by legacy
filter names instead of by filter instance names, but this is expected
to change, as per https://github.com/envoyproxy/envoy/issues/12274; gRPC
will directly implement the desired semantics here.)

When gRPC receives the `RouteConfiguration` proto, either in the LDS or
RDS response, it will validate the contents of each
`typed_per_filter_config` map as follows:
- Every config in the list must be of a known type, as identified by the
  map value proto message type.  Any unknown config type will cause the
  resource to be NACKed.
- The fields in the filter config should be validated by the filter
  implementation.  Any validation error will cause the resource to be NACKed.
- Note that gRPC will *not* fail validation if the map key specifies a
  filter instance name that does not exist in the `HttpConnectionManager`
  filter list.  This is because during an update, the xDS client code
  cannot know which `HttpConnectionManager` config is currently being
  used.
- Similarly, gRPC will *not* fail validation if a map entry uses a
  config for a filter that is supported on only the gRPC client but is
  used on the gRPC server, or vice-versa.  This is because the xDS
  client code cannot know whether a given RDS update will be used by a
  gRPC client or server.

Unlike Envoy, which provides an API that filters must call to access their
per-{VirtualHost,Route,ClusterWeight} config overrides, gRPC will
require filters to construct a merged configuration, applying any
necessary overrides, when the xDS config is applied.  The filters will
then use this merged configuration at run-time.  However, this approach
is an implementation detail that could change in the future.

Note that, for a given request, the filter's configuration will come
from the top-level filter list in the `HttpConnectionManager` config
and the most specific override.  In other words, if a given filter
instance has an override in the `ClusterWeight` proto, that will be
used; otherwise, if it has an override in the `Route` proto, that will
be used; otherwise, if it has an override in the `VirtualHost` proto,
that will be used; otherwise, no override will be used.

### The Router Filter

gRPC will support the [router
filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter),
which is used for the proto message type
[`envoy.extensions.filters.http.router.v3.Router`](https://github.com/envoyproxy/envoy/blob/18db4c90e3295fb2c39bfc7b2ce641cfd6c3fbed/api/envoy/extensions/filters/http/router/v3/router.proto#L23).
As in Envoy, this filter is required and must be last in the filter list.

Note that gRPC will not actually have a discrete filter implementation
for the router filter; instead, this filter will simply trigger the
built-in routing functionality that we have already implemented in the
resolver and LB policy plugins to the gRPC client channel.  However,
in order to retain compatibility with Envoy, gRPC will still accept
this filter configuration and treat it similarly to the way that Envoy
does.

Specifically, if the router filter is not present, gRPC will fail all
requests.  In addition, any filters in the list after the router filter
will not actually be used, although their configurations will still be
validated when the config is received from the xDS server.

For now, gRPC will not support any of the fields in the router filter's
config.  All fields will be ignored.

### Experimental Environment Variable for Initial Testing

On the gRPC client side, this feature will be released at the same time as
the fault injection functionality described in
[gRFC A33](https://github.com/grpc/proposal/pull/201), so it will be
guarded by the same environment variable.  That env var is
`GRPC_XDS_EXPERIMENTAL_FAULT_INJECTION`.  This env var protection will
be removed once the new feature has proven to be stable.

On the gRPC server side, this feature will be released as part of the
initial support for xDS in the gRPC server, as described in
[gRFC A36Servers](https://github.com/grpc/proposal/pull/214).  Because
xDS support in the gRPC server is enabled via the application API, there
is no environment variable guard needed in that case.

## Implementation

C-core implementation:
- Add dynamic filters between name resolution and load balancing: https://github.com/grpc/grpc/pull/24920
- Add xDS HTTP filter support: https://github.com/grpc/grpc/pull/25310
