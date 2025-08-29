A104: xDS Extension Config Discovery Service (ECDS)
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-08-28
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

### Related Proposals: 
* [A39: xDS HTTP Filter Support][A39]
* [A47: xDS Federation][A47]
* [A74: xDS Config Tears][A74]

[A39]: A39-xds-http-filters.md
[A47]: A47-xds-federation.md
[A74]: A74-xds-config-tears.md

## Proposal

When validating the [`HttpConnectionManager.http_filters`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L436),
if the [`config_discovery`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1243)
is set instead of the [`typed_config`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1233),
that indicates that the filter config should be fetched via ECDS.

The ECDS `DiscoveryRequest` will use
[`envoy.config.core.v3.TypedExtensionConfig`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/extension.proto#L20)
as the resource type.  The resource name will come from the [`http_filter.name`
field](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1223).

### LDS Resource Validation Changes

Inside of the `HttpFilter.config_discovery` field, gRPC will look at
the following fields:
- [config_source](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L267):
  This field is ignored.  TODO: should we require this to be a "self"
  ConfigSource?  Not sure that makes sense if we are going to share
  config with Envoy, although that might not be an issue for LDS.
- [default_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L272):
  This field is ignored.
- [apply_default_config_without_warming](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L277):
  This field is ignored.
- [type_urls](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/config/core/v3/config_source.proto#L281C19-L281C28):
  When ECDS returns the `TypedExtensionConfig` resource, it will contain
  an extension of some arbitrary type.  If this field is set, it
  specifies a allowlist of types that are acceptable.  If this list is
  non-empty and the returned extension type is not on the list, then
  the extension will not be accepted.  However, note that we cannot NACK
  the ECDS resource in this case, because it's entirely possible that
  the same ECDS resource will be used in a different filter instance
  where its type *is* in the allowlist.  Instead, we will look at the
  `HttpFilter.is_optional` field: if set to true, then we will skip the
  filter; otherwise, we will instead use a filter that will fail all
  data plane RPCs.

  TODO: should we just not support this field at all, to make this
  failure mode go away?

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
  std::string name;
  std::variant<XdsHttpFilterImpl::FilterConfig, EcdsConfig> config;
};
```

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
example, in C-core, the representation will be the `HttpFilter` struct
shown above.

### gRPC Client-Side Changes

The XdsDependencyManager (see [A74]) will be changed to handle ECDS.
Specifically, when it receives the LDS resource, it will look through the
list of HTTP filters to see if any of them use ECDS.  For those that do,
it will start a watch for the specified ECDS resource.  It will not
return any result to the xds resolver until all ECDS resources have
been obtained.

A new field will be added to the `XdsConfig` struct returned by the
XdsDependencyManager to contain all fetched ECDS resources.  This field
will be a map from resource name to parsed ECDS resource.  The xds
resolver will use that map to get the filter config for any filter in
the LDS resource that uses ECDS.

### gRPC Server-Side Changes

TODO: Figure out what server-side changes are needed

### Composite Filter Changes

TODO: Do we need to support ECDS in the composite filter?

### Temporary environment variable protection

Support for ECDS will be guarded by the `GRPC_EXPERIMENTAL_XDS_ECDS`
environment variable.  This guard will be removed once the feature passes
interop tests.

## Rationale

### Handling Non-Allowlisted Extension Types

Envoy currently lacks a caching XdsClient layer.  As a result, its
documented behavior when the ECDS resource contains a non-allowlisted type
is to reject that update.  That approach won't work for gRPC, because we
*do* have a caching XdsClient layer, and the same resource may be watched
by some other filter instance, where the type *is* in the allowlist.
Therefore, gRPC will either skip the filter or fail data plane RPCs,
depending on the value of `is_optional`.

Note that if we do fail data plane RPCs, this is an instance of a
relatively rare class of config problems where the individual resource
is valid in isolation and therefore cannot be NACKed, but a problem does
appear when we try to combine the resources into a cohesive config.
We do not currently have a good solution for detecting this class of
problems.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
