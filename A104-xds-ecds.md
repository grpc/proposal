A104: xDS Extension Config Discovery Service (ECDS)
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-08-29
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

### Validation of `ExtensionConfigSource` Proto

ECDS is configured via an [`ExtensionConfigSource`
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L266).
The presence of this message will indicate that ECDS should be used, but
gRPC will not actually use any of the fields inside of this proto, each
for different reasons:
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

### LDS Resource Validation Changes

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
  struct EcdsConfig {};

  std::string name;
  std::variant<XdsHttpFilterImpl::FilterConfig, EcdsConfig> config;
};
```

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

TODO: document parsed representation changes

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

The parsed representation of an ECDS resource will be the same as the
representation of an HTTP filter in the parsed LDS resource.  For
example, in C-core, the representation will be the
`XdsHttpFilterImpl::FilterConfig` struct mentioned above.

### gRPC Client-Side Changes

The XdsDependencyManager (see [A74]) will be changed to handle ECDS.
Specifically, when it receives the LDS resource, it will look through the
list of HTTP filters to see if any of them use ECDS.  For those that do,
it will start a watch for the specified ECDS resource.

TODO: don't want XDM to have specific knowledge of the composite filter,
so how will it know to start a watch for the required ECDS resource?

The XdsDependencyManager will not return any result to the xds resolver
until all ECDS resources have been obtained.  If an ECDS resource fails
validation, the XdsDependencyManager will return a transient error to
the xds resolver, which will cause the channel to stick with its
previous good config, if any.

A new field will be added to the `XdsConfig` struct returned by the
XdsDependencyManager to contain all fetched ECDS resources.  This field
will be a map from resource name to parsed ECDS resource.  The xds
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
