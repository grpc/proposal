A104: xDS Extension Config Discovery Service (ECDS)
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-09-04
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC willl support the xDS Extension Config Discovery Service (ECDS),
which allows filter configs to be split into separate xDS resources
rather than being inlined in the LDS resource.

## Background

The xDS HTTP filter support described in [A39] supports filter configs
inlined in the LDS resource.  ECDS is an API whereby the LDS resource
will instead tell the data plane to fetch the config for a given filter
from a separate xDS resource.  This is useful in cases where it is
desirable to get the resource from a different control plane via xDS
federation, as described in [A47].

The ECDS `DiscoveryRequest` uses
[`envoy.config.core.v3.TypedExtensionConfig`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/extension.proto#L20)
as the resource type, and the extension configured within the
`TypedExtensionConfig` dictates what type of filter to use.  This allows
the control plane to dynamically choose the type of the filter.

### Related Proposals:
* [A39: xDS HTTP Filter Support][A39]
* [A47: xDS Federation][A47]
* [A74: xDS Config Tears][A74]
* [A103: xDS Composite Filter][A103] (pending)

[A39]: A39-xds-http-filters.md
[A47]: A47-xds-federation.md
[A74]: A74-xds-config-tears.md
[A103]: https://github.com/grpc/proposal/pull/511

TODO: add Updated-by: header to A103

## Proposal

We will support configuring ECDS in two places:
- In LDS, in the HttpConnectionManager config, as part of the list of
  xDS HTTP filters.
- In the xDS composite filter (see [A103]), as part of the configuration
  of the filter to choose based on the match criteria.

In order to support that second case in a general-purpose way, we will
change the interface for defining xDS HTTP filters such that when a
filter parses its config, it may return an indication of any ECDS
resources that it may require, even transitively.

### Validation of `ExtensionConfigSource` Proto

In both LDS and in the composite filter, ECDS is configured via an
[`ExtensionConfigSource`
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L266).
The presence of this message will indicate that ECDS should be used,
but gRPC will not actually use any of the fields inside of this proto,
each for different reasons:
- [config_source](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L267):
  We will use xDS federation (see [A47]) to determine what server to
  fetch the resource from based on the resource name, so there is no
  need to look at this field.  (In the past, we have required that
  various ConfigSource fields specify the "self" type, but this
  convention predates xDS federation and does not seem to serve any
  value anymore.)
- [default_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L272)
  and
  [apply_default_config_without_warming](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L277):
  We don't currently have any use-case for setting a default config, so
  we will not support these fields.  We can consider adding support for
  these fields in the future if a use-case comes up.
- [type_urls](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L281C19-L281C28):
  We don't have a use-case for restricting the set of filter types that
  may be returned, and supporting this would introduce the possibility
  of a failure mode that cannot be reflected in a NACK, which could
  compromise reliability (see [Rationale](#rationale) below for details).

### xDS HTTP Filter API Changes

As per [A39], when gRPC sees an HTTP filter config in either LDS or RDS,
it will look up the filter implementation in the xDS HTTP filter
registry by the protobuf type of the filter config.  It will then ask that
filter implementation to parse the filter config.

We will change the xDS HTTP filter implementation API such that when it
parses the config, it has the ability to return a set of ECDS resource
names that it depends on.  Note that this will work transitively for
cases like the composite filter: if the composite filter uses the xDS
HTTP filter registry to parse a nested filter config, then any ECDS
resource needed anywhere in that tree will be returned from the
composite filter's parsing function.

Note that in order to avoid potential circular dependencies or stack
overflows, we will impose a maximum recursion depth of 8 when expanding
ECDS dependencies.  This will be done both on the client side (in the
XdsDependencyManager) and on the server side.

TODO: Do we need to take into account the full height of the HTTP
filter config stack, even if some are inlined and others use ECDS?  Or
are we okay with imposing separate limits on inline and ECDS resolution,
so that max is really the product of the two?

### LDS Resource Validation Changes

We will add a new field to the parsed representation of the
`HttpConnectionManager` config that will indicate the set of ECDS resource
names that are needed as dependencies.  For example, in C-core, it will
look like this:

```
std::set<std::string> ecds_resources_needed;
```

When validating the entries in the [`HttpConnectionManager.http_filters`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L436),
if the [`config_discovery`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1243)
is set instead of the [`typed_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1233),
that indicates that the filter config should be fetched via ECDS.
In this case, the ECDS resource name comes from the [`http_filter.name`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1223).

The parsed LDS resource will need to represent ECDS configuration for
individual filters.  For example, in C-core, the representation of a
filter today looks like this:

```c++
struct HttpFilter {
  std::string name;
  XdsHttpFilterImpl::FilterConfig config;
};
```

We will change this to instead look like this:

```c++
struct HttpFilter {
  struct UseEcds {};

  std::string name;
  std::variant<XdsHttpFilterImpl::FilterConfig, UseEcds> config;
};
```

If the `config` field contains a `UseEcds` struct, that means that the
config for the filter should be obtained from an ECDS resource with name
from the `name` field.  In this case, the name of the ECDS resource will
be added to the `ecds_resources_needed` field in the parsed
`HttpConnectionManager` config.

If the `config` field contains an inline filter config rather than a
`UseEcds` struct, then we will use the xDS HTTP filter registry to parse
the specified filter config, just like we do today.  In this case, any
ECDS resources needed that the filter parsing returned will be added to
the `ecds_resources_needed` field in the parsed `HttpConnectionManager`
config.

Note that when a filter's configuration is obtained from a separate ECDS
resource, that means that the `HttpConnectionManager` validation code
will not know the type of the filter and therefore will not know whether
the filter is a terminal filter.  To avoid causing problems after config
validation time, we will support ECDS only for non-terminal filters.
This means that in the LDS validation logic, it will be considered an
error if the last filter in the chain (which must be terminal) uses ECDS
instead of an inline config.

### RDS Resource Validation Changes

Just like in LDS, we will add a new `ecds_resources_needed` field to
the parsed representation of the `RouteConfiguration` resource that will
indicate the set of ECDS resource names that are needed as dependencies.
This field will have the same type as the corresponding field described
in the `HttpConnectionManager` config above.

As described in [xDS HTTP Filter API Changes](#xds-http-filter-api-changes)
above, as we use the xDS HTTP filter registry to parse the configs in
the various `typed_per_filter_config` fields, the xDS HTTP filter
implementation will return a list of ECDS resource names that it depends
on.  This list will be added to the `ecds_resources_needed` field in the
parsed `RouteConfiguration` resource.

### xDS Composite Filter Validation Changes

As per [A103], the xDS composite filter is configured using a
[`ExecuteFilterAction`
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L49).
In that proto, if the [`dynamic_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L59)
is set instead of the [`typed_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L54),
that indicates that the filter config should be fetched via ECDS.
In this case, the ECDS resource name comes from the [`DynamicConfig.name`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L39).

The validation rules for the fields in the `ExecuteFilterAction` proto
will now be (first match wins):
- If `dynamic_config` is set, that gets used.
- If `filter_chain` is set, that gets used.
- If `typed_config` is set, that gets used.
- Otherwise, the config is considered invalid.

Note that `dynamic_config` is being set as higher precedence to the other
fields, because gRPC will already support the other two fields, so this
provides a path for control planes to migrate to the new field without
breaking existing gRPC clients.  However, control plane operators should
be aware that Envoy currently requires that exactly one of `typed_config`
or `dynamic_config` are present, so a config setting both of those is
likely to be rejected by Envoy.

The parsed representation of the composite filter will change in a similar
way to the parsed LDS representation.  As per [A103], the actions in the
matcher tree can have two possible values, a parsed filter config or an
indication to skip the filter.  With the addition of ECDS support, a
third possible value will be added, which is the ECDS resource name to
fetch.

### ECDS Resource Validation

We will implement a new xDS resource type for `TypedExtensionConfig`
resources.  The [`name`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/extension.proto#L23)
will indicate the resource name when not wrapped in a [`Resource`
wrapper](https://github.com/envoyproxy/envoy/blob/ee9d3b13f0e73930236b9371554e3f3ac3fcf7c4/api/envoy/service/discovery/v3/discovery.proto#L386).

Although in principle, ECDS allows fetching extensions of *any* type,
gRPC will initially support it only for HTTP filters.  As a result, we
can hard-code the use the xDS HTTP filter registry when validating ECDS
resources.  We will validate the contents of the [`typed_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/extension.proto#L31)
via the xDS HTTP filter registry, just as we do today for the
[`HttpFilter.typed_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1233).

As mentioned above, we support ECDS for only non-terminal filters.
Therefore, if the filter implementation for the filter type indicates
that it is a terminal filter, the ECDS resource will be NACKed.

The parsed representation of an ECDS resource will have two fields in it:
- The parsed representation of the enclosed filter config, in the same
  form as encoded in the parsed `HttpConnectionManager` config.
- A list of dependent ECDS resources.

As an example, in C-core, the ECDS parsed representation will look
something like this:

```c++
struct XdsEcdsResource {
  XdsHttpFilterImpl::FilterConfig config;
  std::set<std::string> ecds_resources_needed;
};
```

### gRPC Client-Side Changes

The XdsDependencyManager (see [A74]) will be changed to handle ECDS.
ECDS resources will be added to the dependency tree in the following
cases:
- From the `ecds_resources_needed` field in a parsed LDS resource.
- From the `ecds_resources_needed` field in a parsed
  `RouteConfiguration` resource (regardless of whether it is inlined
  into LDS or obtained via RDS).
- From the `ecds_resources_needed` field in a parsed ECDS resource.

Whenever it sees an update that changes the set of ECDS resources
needed, it will start watches for any ECDS resource that it is not
already watching, and it will stop watches for any ECDS resource that it
had been watching but that is no longer referenced by the configuration.

The XdsDependencyManager will not return any result to the xds resolver
until all ECDS resources have been obtained.  If an ECDS resource returns
a data error, the XdsDependencyManager will return an error to
the xds resolver, just as if the LDS or RDS resources had seen the
error.

A new field will be added to the `XdsConfig` struct returned by the
XdsDependencyManager to contain all fetched ECDS resources.  This field
will be a map from resource name to the parsed ECDS resource.  The xds
resolver will use that map to get the filter config for any filter in
the LDS resource that uses ECDS.

### gRPC Server-Side Changes

TODO: Figure out what server-side changes are needed
(should have essentially the same behavior as XDM on client side)

### Temporary environment variable protection

Support for ECDS will be guarded by the `GRPC_EXPERIMENTAL_XDS_ECDS`
environment variable.  This guard will be removed once the feature passes
interop tests.

## Rationale

We will not support the `ExtensionConfigSource.type_urls` field, because
that would introduce the possibility of a failure without a NACK to inform
the control plane of the problem.  The XdsClient needs to decide whether
to NACK when it first recieves the ECDS resource and checks whether
it's valid, and only if it is will it update its cache and report it
to the watchers.  However, only the watchers know the context in which
the ECDS resource is being requested, which is where the allowlist of
filter types would be configured.  It's entirely possible that the same
ECDS resource could be used in two different filter instances, where
its type is allowed in one but disallowed in another, and we wouldn't
discover that until we send the update to the watchers.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
