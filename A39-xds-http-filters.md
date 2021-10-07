A39: xDS HTTP Filter Support
----
* Author(s): Mark D. Roth (markdroth)
* Approver: ejona86, dfawley
* Status: Approved
* Implemented in: C-core
* Last updated: 2021-07-20
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

### HTTP Filter Registry

gRPC will have a filter registry that tells it the set of supported
xDS HTTP filters.

Each filter will be registered by the protobuf message type(s) that
represent its configuration in xDS.  Note that every filter has a
message type for its top-level configuration, but not all filters
support override configuration at all, and those that do may or may not
use the same message type for their override configuration as for their
top-level configuration.  Therefore, every filter will be registered
by one or more message types.

Each filter implementation will provide the following:
- Method(s) for validating the config protos when parsing an xDS response.
- An indication of whether it is supported on the gRPC client and/or gRPC
  server.
- An indication of whether the filter is a terminal filter (i.e., it
  must be the last filter in the filter chain).  (Note that the
  [router filter](#the-router-filter) is currently the only terminal
  filter supported by gRPC, so implementations may choose to hard-code
  the requirement that that filter is the last filter in the chain
  instead of supporting a general-purpose mechanism to determine whether
  a given filter is a terminal filter.)
- Methods to generate and configure the appropriate filters or
  interceptors in gRPC to perform the necessary functionality on the
  data plane.

### xDS API Fields

gRPC will add support for the following fields in the xDS APIs:
- [`HttpConnectionManager.http_filters`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L271)
- [`VirtualHost.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/config/route/v3/route_components.proto#L144)
- [`Route.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/config/route/v3/route_components.proto#L257)
- [`WeightedCluster.ClusterWeight.typed_per_filter_config`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/config/route/v3/route_components.proto#L374)

gRPC will support configuring multiple instances of the same filter
implementation, each with its own name and config.  (Envoy supports this
in the `HttpConnectionManager` config but does not yet support per-route
config overrides in such cases, as per
https://github.com/envoyproxy/envoy/issues/12274.)

gRPC will also support the ability to indicate that a given filter
is optional, as recently added in
https://github.com/envoyproxy/envoy/pull/14982 and implemented in
Envoy in https://github.com/envoyproxy/envoy/pull/16119.

#### Top-Level Filter Configs

The `HttpConnectionManager.http_filters` field is an ordered list of
filters.  In each entry, gRPC will look at the following fields:
- [`name`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L831):
  A logical name for the filter instance.  This name will be used when
  applying overrides from the virtual host, route, or weighted cluster entry.
  The logical instance name also allows configuring multiple instances of the
  same filter, which can be useful for cases like authorization policies.
- [`typed_config`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L836):
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
- [`is_optional`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L836):
  If set to true, if the filter config type is unknown, then the filter
  will be ignored instead of causing the resource to be NACKed.

When gRPC receives the `HttpConnectionManager` proto in the LDS
response, it will validate the list of filters as follows:
- There must be at least one filter in the list, or the Listener
  resource will be NACKed.
- Every filter in the list must have a unique instance name.  Any
  duplicate instance names will cause the Listener resource to be NACKed.
- Every filter config in the list must be of a type (as identified by the
  `type_url` field) that is present in the filter registry.  Note that the
  set of known filter config types will be different on the gRPC client and
  server; if the `HttpConnectionManager` proto is inside an HTTP API Listener,
  it will look only at filters registered for the gRPC client, whereas if it
  is inside a TCP Listener, it will look only at filters registered for
  the gRPC server.  Any unknown config type will cause the Listener
  resource to be NACKed, unless the `is_optional` field is true, in which
  case the individual filter will be ignored.
- If the filter config type is present in the filter registry (i.e., we
  have not NACKed or ignored the entry, depending on the value of
  `is_optional`), the fields in the filter config will be validated by
  the filter implementation.  The filter implementation will also verify
  that the config is of the right type (e.g., if it uses different types
  for its top-level config and its override config).  In addition,
  validation will fail if a terminal filter is not the last filter in
  the chain or if a non-terminal filter is the last filter in the chain.
  (Note that the only currently supported terminal filter is the [router
  filter](#the-router-filter).)  Any validation error will cause the
  Listener resource to be NACKed.

#### Filter Config Overrides

In each of the `VirtualHost`, `Route`, and `WeightedCluster.ClusterWeight`
protos, there is a field called `typed_per_filter_config` that contains
per-filter overrides.  Each entry in the map will be used as follows:
- The entry's key is the filter instance name to which the config override
  applies.  The entry will be used for the filter in the
  `HttpConnectionManager.http_filters` list with the same `name` field.
- The entry's value contains the override config for the filter.  As with
  the `HttpConnectionManager.http_filters.typed_config` field, the type
  of the protobuf message determines which filter is used, based on the
  filters known to the filter registry.
- If the value protobuf message type is `envoy.config.route.v3.FilterConfig`,
  then gRPC will look inside that type's
  [`config`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/config/route/v3/route_components.proto#L1966)
  field to determine the actual type.  In addition, gRPC will look at the
  [`is_optional`](https://github.com/envoyproxy/envoy/blob/9ed95162ce11a09a56c1743d0a6e342d9bbfd2b6/api/envoy/config/route/v3/route_components.proto#L1971)
  field to determine how to handle unknown protobuf message types.
- Whether or not the `FilterConfig` wrapper is used, if the type is
  `udpa.type.v1.TypedStruct`, then gRPC will look inside that type's
  [`type_url`](https://github.com/cncf/udpa/blob/cc1b757b3eddccaaaf0743cbb107742bb7e3ee4f/udpa/type/v1/typed_struct.proto#L38)
  field to determine the actual type; this allows for third-party
  plugins to encode their configurations as a `google.protobuf.Struct`
  proto instead of using a concrete proto message type that the client
  may not be able to handle.

Note that Envoy currently keys these maps by legacy filter names instead
of by filter instance names, but this is expected to change, as per
https://github.com/envoyproxy/envoy/issues/12274.  gRPC will directly
implement the desired semantics here.

When gRPC receives the `RouteConfiguration` proto, either in the LDS or
RDS response, it will validate the contents of each
`typed_per_filter_config` map as follows:
- Every config in the list must be of a known type, as identified by the
  map value proto message type.  Any unknown config type will cause the
  resource to be NACKed, unless the `is_optional` field is true, in which
  case the individual entry will be ignored.
- If the filter config type is present in the filter registry (i.e., we
  have not NACKed or ignored the entry, depending on the value of
  `is_optional`), the fields in the filter config will be validated by
  the filter implementation.  The filter implementation will also verify
  that the config is of the right type (i.e., it will fail validation if
  the override config is of the filter's top-level config type and
  either the filter does not support an override config or it uses
  different types for its top-level config and its override config).
  Any validation error will cause the resource to be NACKed.
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
This filter is a terminal filter, and it is currently the only terminal
filter supported by gRPC, so it must be the last one in the filter chain.
This will be enforced at config validation time, as described above.

Note that gRPC will not actually have a discrete filter implementation
for the router filter; instead, this filter will simply trigger the
built-in routing functionality that we have already implemented in the
resolver and LB policy plugins to the gRPC client channel.  However,
in order to retain compatibility with Envoy, gRPC will still accept
this filter configuration and treat it similarly to the way that Envoy
does.

For now, gRPC will not support any of the fields in the router filter's
config.  All fields will be ignored.

### Experimental Environment Variable for Initial Testing

On the gRPC client side, this feature will be released at the same time as
the fault injection functionality described in
[gRFC A33](https://github.com/grpc/proposal/pull/201), so it will be
guarded by the same environment variable.  That env var is
`GRPC_XDS_EXPERIMENTAL_FAULT_INJECTION`.

On the gRPC server side, this feature will be released as part of the
initial support for xDS in the gRPC server, as described in
[gRFC A36](https://github.com/grpc/proposal/pull/214), where the
first feature is mTLS support, which is guarded by the
`GRPC_XDS_EXPERIMENTAL_SECURITY_SUPPORT` env var.

Note that since the xDS resource validation code is shared between the
gRPC client and server, the relevant xDS fields will be processed
whenever either of these environment variables are set.

This env var protection will be removed once this new feature has proven to
be stable.

## Implementation

C-core implementation:
- Add dynamic filters between name resolution and load balancing: https://github.com/grpc/grpc/pull/24920
- Add xDS HTTP filter support for gRPC client: https://github.com/grpc/grpc/pull/25310
- Add support for filters being marked as terminal: https://github.com/grpc/grpc/pull/26742
