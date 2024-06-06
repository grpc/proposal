A83: xDS GCP Authentication Filter
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-05-21
* Discussion at: https://groups.google.com/g/grpc-io/c/76a0zWJChX4

## Abstract

In service mesh environments, there are cases where intermediate proxies
make it impossible to rely on mTLS for end-to-end authentication.  These
cases can be addressed instead by the use of JWT identity tokens.  The
xDS [GCP Authentication
filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/gcp_authn_filter)
provides a mechanism for attaching such JWT tokens as gRPC call
credentials on GCP.  We will add support for this filter in gRPC.

## Background

gRPC already supports a framework for xDS HTTP filters, as described in
[gRFC A39][A39].  We will support the GCP Authentication filter under
this framework.

### Related Proposals: 
* [gRFC A39: xDS HTTP Filters][A39]
* [gRFC A60: xDS-Based Stateful Session Affinity for Weighted Clusters][A60]
* [RFC-7519: JSON Web Token (JWT)][RFC-7519]

[A39]: A39-xds-http-filters.md
[A60]: A60-xds-stateful-session-affinity-weighted-clusters.md
[RFC-7519]: https://datatracker.ietf.org/doc/html/rfc7519

## Proposal

We will support the GCP Authentication xDS HTTP filter in the gRPC client.

### xDS HTTP Filter Configuration

The xDS HTTP filter will be configured via the
[`extensions.filters.http.gcp_authn.v3.GcpAuthnFilterConfig`
message](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/extensions/filters/http/gcp_authn/v3/gcp_authn.proto#L24).
The fields will be interpretted as follows:
- `http_uri`: Required.  If not present, the config will be NACKed.
  Within this message:
  - `uri`: Required.  If not present, the config will be NACKed.
  - `timeout`: Required.  If not present, the config will be NACKed.
    This field will be validated the same way as any
    `google.protobuf.Duration` message in xDS; specifically:
    - `seconds`: Must be from 0 to 315,576,000,000 inclusive.  If the
      value is outside that range, the config will be NACKed.
    - `nanos`: Must be from 0 to 999,999,999 inclusive.  If the value
      is outside that range, the config will be NACKed.
  - `cluster`: This field will be ignored by gRPC.  (This field is
    required by Envoy, but gRPC will not use it.)
- `cache_config`: Optional.  Within this message:
  - `cache_size`: Must be between 0 and `INT64_MAX`.  0 disables the cache.
- `token_header`: Optional.  Within this message:
  - `name`: Required; if not present, the config will be NACKed.  The name
    of the header to set.
  - `value_prefix`: Optional.  The prefix to add before the JWT token in
    the header value.
- `retry_policy`: For now, we will not support this field, and there will
  be no retries for HTTP requests.  If the lack of retries becomes a
  problem, we can consider adding this later.

### xDS Cluster Metadata

The GCP Authentication filter uses cluster metadata from the
[`Cluster.metadata`
field](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/config/cluster/v3/cluster.proto#L1092)
to configure the audience.  We will process this field when validating
the CDS resource and convert it into a map, which will be added to
the parsed cluster resource that is passed to the XdsClient watcher.

The metadata field is a message that actually contains two maps:
- [`filter_metadata`](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/config/core/v3/base.proto#L248):
  This map contains `google.protobuf.Struct` values, which we will
  convert to parsed JSON form.
- [`typed_filter_metadata`](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/config/core/v3/base.proto#L257):
  This map contains `google.protobuf.Any` fields.  To support this, we
  will use a registry-like approach for metadata types (may be an actual
  registry or just a block of code that supports the known protobuf
  message types) that handles parsing the `google.protobuf.Any` field
  and converting it some internal form appropriate for the implementation
  (e.g., a native struct).

The value for a given metadata key will come from only one of the
two maps; the value from `filter_config` will be used only if the
key is not present or is of an unknown protobuf message type in
`typed_filter_config`.  In the resulting map in the parsed cluster
resource, the map value will be either JSON or the appropriate internal
form, depending on which of the two maps the entry came from.

To match Envoy, the logic to validate cluster metadata will look something
like this (pseudo-code):

```
parsed_metadata = {}  # Value is either JSON or parsed object
all_keys = (cluster_metadata.typed_filter_metadata.keys() +
            cluster_metadata.filter_metadata.keys())
for key in all_keys:
  # First try typed_filter_metadata.
  any_field = cluster_metadata.typed_filter_metadata[key]
  if any_field is not None:
    parser = metadata_registry.FindParser(any_field.type_url)
    if parser is not None:
      value = parser.Parse(any_field.value)
      if value is None:
        return NACK  # Parsing failed, reject resource
      parsed_metadata[key] = value
      continue
    # If no parser registered for this type, ignore the entry.
  # Didn't find it in typed_filter_metadata, so try filter_metadata.
  struct_field = cluster_metadata.filter_metadata[key]
  if struct_field is not None:
    parsed_metadata[key] = ConvertToJson(struct_field)
```

For now, the only registered metadata type we support is
[`extensions.filters.http.gcp_authn.v3.Audience`](https://github.com/envoyproxy/envoy/blob/c10da96eae1be169a14772480a61a6316df12edd/api/envoy/extensions/filters/http/gcp_authn/v3/gcp_authn.proto#L42).
In this message, the `url` field must be non-empty; if empty, the
resource will be NACKed.  The parsed representation of this message can
be a simple string.

Note that in Envoy, the metadata keys must exactly match the legacy
filter name (e.g., "envoy.filters.http.gcp_authn").  However, as per
envoyproxy/envoy#34251, it is desirable to instead use the HTTP filter
instance name from the [`HttpFilter.name`
field](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1149).
We will implement that behavior in gRPC.

As per [gRFC A60][A60], we currently pass the selected cluster name via
a call attribute for access in filters.  We will now also pass the CDS
resource for the cluster in a separate call attribute, so that the GCP
Authentication filter can access the cluster metadata for the selected
cluster.

### Filter Behavior

When the filter processes the RPC's initial metadata, it will first
attempt to determine the audience for the request by looking at the
cluster metadata key corresponding to the filter instance name, which
must be of type `extensions.filters.http.gcp_authn.v3.Audience`.  If the
cluster has no such metadata key, the filter is a no-op.  Otherwise, the
audience is the value of the `url` field in the `Audience` proto.

The filter will then check to see if it already has a cached JWT token
for the specified audience.  If so, it will add the token to the RPC's
headers as described below.

If the JWT token is not cached, then the filter will send an HTTP request
to fetch the JWT token.  The URI of the request will come from the filter
config's `http_uri.uri` field, substituting the string `[AUDIENCE]` with
the actual audience.  The HTTP request will include the header
`Metadata-Flavor: Google`.  The HTTP request will have a deadline set
from the `http_uri.timeout` field in the filter config.

If the HTTP request succeeds, the body of the response will contain
the JWT token, which the client will both add to its cache and add
to the data plane RPC's headers.  The client does not need to do full
[RFC-7519] validation of the token (that is the responsibility of the
server side), but it does need to extract the `nbf` and `exp` fields
for caching purposes.

If the HTTP request fails or times out, or if the JWT token is invalid
(i.e., the client fails to extract the `nbf` or `exp` fields), the data
plane RPC will be sent without the JWT token and will then fail on the
destination server if authentication was required.

If a JWT token is obtained either from the cache or from an HTTP
request, it will be added to the RPC headers.  If the filter config has
no `token_header` field, then the header name will be `authorization`
and the value prefix will be `Bearer ` (note trailing space); otherwise,
the header name and value prefix from `token_header` will be used.

### Temporary environment variable protection

Support for the GCP Authentication filter in LDS will be guarded by the
`GRPC_EXPERIMENTAL_XDS_GCP_AUTHENTICATION_FILTER` env var.  The env var
guard will be removed once the feature passes interop tests.

## Rationale

It is not our intention to support this mechanism for GCP only; in
principle, it should be possible to support JWT identity tokens for any
cloud provider.  However, at present, the existing xDS HTTP filter
supports only GCP, so that's what we're initially focusing on, for
compatibility with Envoy.  We would be open to future contributions from
the OSS communicate to provide similar functionality for other cloud
providers, in both gRPC and Envoy.

## Implementation

Will be implemented in all languages, timelines TBD.
