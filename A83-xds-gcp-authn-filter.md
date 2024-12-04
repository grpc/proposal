A83: xDS GCP Authentication Filter
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-12-04
* Discussion at: https://groups.google.com/g/grpc-io/c/76a0zWJChX4

## Abstract

In service mesh environments, there are cases where intermediate proxies
make it impossible to rely on mTLS for end-to-end authentication.  These
cases can be addressed instead by the use of service account identity
JWT tokens.  The xDS [GCP Authentication
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
* [gRFC A74: xDS Config Tears][A74]
* [RFC-7519: JSON Web Token (JWT)][RFC-7519]

[A39]: A39-xds-http-filters.md
[A60]: A60-xds-stateful-session-affinity-weighted-clusters.md
[A74]: A74-xds-config-tears.md
[RFC-7519]: https://datatracker.ietf.org/doc/html/rfc7519

## Proposal

We will support the GCP Authentication xDS HTTP filter in the gRPC client.

### Call Credentials

Note: This section is intended for gRPC implementations that need to
implement a new call credential type for GCP service account identity
tokens.  Implementations that already support this functionality (e.g.,
by depending on an external Google Auth library) may continue to use
their existing functionality, even if the behavior differs in small ways
from what is described in this section.

gRPC should support a GcpServiceAccountIdentityCallCredentials call
credentials type, which is not xDS-specific.  This credential type will
be instantiated with one parameter, which is the audience to be encoded
into the JWT token.  The credential object will handle fetching the
token on-demand and caching it based on the token's expiration time.

To handle potential clock skew issues and to account for processing time
on the server, the credential will set the cache expiration time to be
30 seconds before the expiration time encoded in the token.  All logic
in the call credential code will use this modified expiration time
instead of the expiration time encoded in the token.

When the credential is asked for a token for a data
plane RPC, if the token is not yet cached or the cached
token will expire within some fixed refresh interval
(typically 1 minute), the credential will start an HTTP request (if there
is not already one pending) to
`http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=[AUDIENCE]`,
where `[AUDIENCE]` is replaced with the audience specified when the
credential object was instantiated.  The HTTP request will include the
header `Metadata-Flavor: Google`.

When a data plane RPC starts, if the token is cached and is not expired,
the token will immediately be added to the RPC, and the RPC will continue.
Otherwise (i.e., before the token is initially obtained or after the
cached token has expired), the data plane RPC will be queued until the
HTTP request completes.  When the HTTP request completes, the result
(either success or failure, as described below) will be applied to all
queued data plane RPCs.

Note that when the token's expiration time is less than the refresh
interval in the future, a new data plane RPC being started will trigger
a new HTTP request, but the cached token value will still be used for
that data plane RPC.  This pre-emptive re-fetching is intended to avoid
periodic latency spikes when refreshing the token.

If the HTTP request fails, all queued data plane RPCs will be failed
with a gRPC status determined based on the returned HTTP status.  If the
returned HTTP status maps to `UNAVAILABLE` in [HTTP to gRPC Status Code
Mapping](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md),
then the data plane RPCs will be failed with status `UNAVAILABLE`;
otherwise, they will be failed with status `UNAUTHENTICATED`.  If the
request fails without an HTTP status (e.g., an I/O error), all queued
data plane RPCs will be failed with `UNAVAILABLE` status.

If the HTTP request succeeds, the body of the response will contain the
JWT token. which the client will cache.  The client does not need to
do full [RFC-7519] validation of the token (that is the responsibility
of the server side), but it does need to extract the `exp` field for
caching purposes.  If the `exp` field cannot be extracted (i.e., the JWT
token is invalid), all queued data plane RPCs will be failed with status
`UNAUTHENTICATED`.  Otherwise, the cache is updated, and the returned
token is added to all queued data plane RPCs, which may then continue.

If the HTTP request does not result in the cache being updated (i.e.,
if the HTTP request fails or if it returns an invalid JWT token),
[backoff](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
must be applied before the next attempt may be started.  If a data
plane RPC is started when there is no cached token available and while
in backoff delay, it will be failed with the status from the last HTTP
request attempt.  When the backoff delay expires, the next data plane
RPC will trigger a new attempt.  Note that no attempt should be started
until and unless a data plane RPC is started, since we do not want to
unnecessarily retry if the channel is idle.  The backoff state will be
reset once there is a successful HTTP request.

To add the token to a data plane RPC, the call credential will add a
header named `authorization`.  The header value will be the string
`Bearer ` (note trailing space) followed by the token value.

### xDS HTTP Filter Configuration

The xDS HTTP filter will be configured via the
[`extensions.filters.http.gcp_authn.v3.GcpAuthnFilterConfig`
message](https://github.com/envoyproxy/envoy/blob/c16faca3619fb44c24b12d15aad8a797b9e210ab/api/envoy/extensions/filters/http/gcp_authn/v3/gcp_authn.proto#L27).
The fields will be interpretted as follows:
- `cache_config`: Optional.  Within this message:
  - `cache_size`: Optional.  If set, must be greater than 0.  Defaults
    to 10.  Implementations that cannot support caches as large as
    `UINT64_MAX` may cap this value at their maximum supported size.
- `http_uri`: Ignored by gRPC.
- `token_header`: Ignored by gRPC.
- `retry_policy`: Ignored by gRPC.
- `cluster`: Ignored by gRPC.
- `timeout`: Ignored by gRPC.

Note that this filter does not support having its config overridden in a
`typed_per_filter_config` field on a per-route, per-virtualhost, or
per-clusterweight basis.  If the filter's config message appears in a
`typed_per_filter_config` field, it will be validated as part of the
normal resource validation, but the configuration will not actually be
used.

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
  (e.g., JSON or a native struct).

The value for a given metadata key will come from only one of the
two maps; the value from `filter_metadata` will be used only if the
key is not present or is of an unknown protobuf message type in
`typed_filter_metadata`.  In the resulting map in the parsed cluster
resource, the map value will contain the type of the original message
(`google.protobuf.Struct` if it came from the `filter_metadata` map) and
a parsed representation of the content.  The parsed representation may
be either JSON or the appropriate internal form, depending on which of
the two maps the entry came from.

The logic to validate cluster metadata will look something like this
(pseudo-code):

```python
parsed_metadata = {}  # Value is either JSON or parsed object
# First process typed_filter_metadata.
for key, any_field in cluster_metadata.typed_filter_metadata.items():
  parser = metadata_registry.FindParser(any_field.type_url)
  if parser is not None:
    value = parser.Parse(any_field.value)
    if value is None:
      return NACK  # Parsing failed, reject resource
    parsed_metadata[key] = value
# Now process filter_metadata.  We look only at keys that were not
# already added from typed_filter_metadata.
for key, struct_field in cluster_metadata.filter_metadata.items():
  if key not in parsed_metadata:
    parsed_metadata[key] = ConvertToJson(struct_field)
```

For now, the only registered metadata type we support is
[`extensions.filters.http.gcp_authn.v3.Audience`](https://github.com/envoyproxy/envoy/blob/c16faca3619fb44c24b12d15aad8a797b9e210ab/api/envoy/extensions/filters/http/gcp_authn/v3/gcp_authn.proto#L66).
In this message, the `url` field must be non-empty; if empty, the
resource will be NACKed.  The parsed representation of this message can
be a simple string.

### xDS ConfigSelector Behavior

As per [gRFC A60][A60], we currently pass the selected cluster name via
a call attribute for access in filters.  However, the filters will now
also need access to the CDS resource for the selected cluster, so that
the GCP Authentication filter can access the cluster metadata for the
selected cluster.  This data is available via the `XdsConfig` attribute
introduced in [A74].  If the xDS ConfigSelector is not already passing
that attribute to the filters, it will need to be changed to do so.

### Filter Call Credentials Cache

The filter will maintain a cache of
GcpServiceAccountIdentityCallCredentials instances, one for each audience,
along with a last-used list that tracks how recently the entries
have been used.  As an entry is used, it is moved to the front of the
last-used list.  The maximum number of entries in the cache is bounded
by the config field `cache_config.cache_size`; if the cache exceeds that
size, then entries will be removed starting from the end of the
last-used list.

If an LDS update changes the cache size, the filter must apply that
change to the cache.  If the cache currently has more entries in it than
the new cache size, then the least recently used entries will be removed
to make the cache adhere to the new size limit.

#### Cache Sharing and Lifetime

Note that the `cache_config.cache_size` parameter in the filter config
is a channel-level parameter, not settable per-route, and we want the
cache itself to be shared across all routes.  Implementations that create
separate filter/interceptor instances for each route should share the
cache between those instances.

It is desirable to avoid losing this cache when we get an xDS Listener or
RouteConfiguration update, so that we don't wind up needlessly refetching
tokens after the update.  Implementations should provide a mechanism for
new instances of the filter to retain the cache from previous instances.

To address these requirements, we will implement a general-purpose
mechanism for xDS HTTP filters to retain state across updates.  The GCP
Authentication filter will be the first use of this mechanism, but
we expect this mechanism will be used by other filters in the future,
such as the upcoming server-side rate limiting filter.

Due to the different nature of xDS HTTP filter support in C-core vs. the
other implementations, the details here differ by language.

##### C-core

In C-core, we introduce a mechanism called a "blackboard" to allow
filters to set arbitrary state that can be accessed by subsequent filter
instances.

The key for each blackboard entry is the unique type of the value and
a string that identifies the instance of that type.  The unique type
will be different for each type of data stored, which avoids conflicts
between different xDS HTTP filter implementations.  The string
differentiates between different instances of the same type, which can
be used to share data between filter instances -- e.g., if a filter uses
its configuration as the key string, then two instances of that filter
that share the same config will use the same instance.

Entries in a blackboard are ref-counted.  Whenever we create a new
filter stack (i.e., upon receiving an LDS or RDS update), each filter
has access to both the previous blackboard (if any) and to a new
blackboard, which starts empty.  As each filter is initialized, it can
look for entries in the old blackboard to reuse, and any such entry is
added to the new blackboard.  The channel then destroys the old
blackboard and replaces it with the new one, which it will retain until
the next time it creates a new filter stack.

The GCP Authentication filter will use this mechanism for the call
credentials cache.  The blackboard key string will be the filter's
instance name, so that two instances of this filter will not share
their cache, but the same instance will retain its cache instance
across updates.

The cache will provide a `SetMaxSize()` method, so that if a config
update changes the cache size, we can apply that change to the existing
cache without having to lose the cache contents.  Because this cache
object is shared between the old and new filter stacks during an update,
a cache size change will wind up affecting the old filter instance,
which in principle it shouldn't, but that is considered acceptable for
this type of change.

##### Java and Go

TODO(sergiitk, ejona86, dfawley): Fill this in.

### Filter Behavior

When the filter processes the RPC's initial metadata, it will first
check to see what cluster the RPC is being sent to.  If the RPC is being
sent to a route that uses a cluster specifier plugin instead of a fixed
cluster, then the filter is a no-op.  Otherwise, the filter will attempt
to determine the audience by looking at the CDS resource for the cluster
that the RPC is being sent to.

If the CDS resource is not available (e.g., because the client received an
error without having previously received a valid resource, or because the
server indicated that the resource has been deleted), then the filter will
fail the RPC with status `UNAVAILABLE`.  Note that this does yield
sub-optimal behavior for wait_for_ready RPCs, since we will fail them
instead of queuing them, but we don't currently have a good alternative:
the filter cannot queue the call until the client gets a valid CDS
resource, because once that happens, a new instance of the filter will be
swapped in for subsequent calls, but the queued call would already be tied
to the original filter instance, which will never see the update.

Otherwise, the filter will look in the CDS resource's metadata for
a key corresponding to the filter's instance name.  Note that
in Envoy, the cluster metadata keys must exactly match the
legacy filter name (e.g., "envoy.filters.http.gcp_authn").
However, as per envoyproxy/envoy#34251, it is desirable
to instead use the HTTP filter instance name from the [`HttpFilter.name`
field](https://github.com/envoyproxy/envoy/blob/7436690884f70b5550b6953988d05818bae3d087/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1149).
We will implement that behavior in gRPC.

If the cluster metadata does not contain a key matching the filter's
instance name, then the filter is a no-op.  If a cluster metadata entry
exists for the filter's instance name, but the entry is of a type other
than `extensions.filters.http.gcp_authn.v3.Audience`, then the filter
will fail data plane RPCs with status `UNAVAILABLE`.  Otherwise, the
audience is the value of the `url` field in the `Audience` proto.

The filter will then check to see if it already has a cached
GcpServiceAccountIdentityCallCredentials instance for the specified
audience.  If it does not, it will create a new instance, adding it to
its cache, removing the least recently used entry from the cache if the
cache is already at its max size.  It will then attach that
GcpServiceAccountIdentityCallCredentials instance to the RPC.

Note that implementations must ensure that the token is not added to
RPCs sent on insecure connections.  However, the GCP Authentication
filter will run before load balancing has chosen a connection, so the
filter cannot directly add the token to the RPC.  Instead, it must add
the call credential to the RPC, and the call credential will do the work
of adding the token to the RPC later, after load balancing has chosen a
connection.

### Temporary environment variable protection

Support for the GCP Authentication filter in the xDS HTTP filter
registry and the `extensions.filters.http.gcp_authn.v3.Audience`
entry in the metadata registry will be guarded by the
`GRPC_EXPERIMENTAL_XDS_GCP_AUTHENTICATION_FILTER` env var.  The env var
guard will be removed once the feature passes interop tests.

## Rationale

It is not our intention to support this mechanism for GCP only; in
principle, it should be possible to support JWT identity tokens for any
cloud provider.  However, at present, the existing xDS HTTP filter
supports only GCP, so that's what we're initially focusing on, for
compatibility with Envoy.  We would be open to future contributions from
the OSS community to provide similar functionality for other cloud
providers, in both gRPC and Envoy.

Note that the cache structure in this design is a bit different from
Envoy's implementation.  In Envoy, the GCP Authentication filter directly
maintains a single cache containing the tokens for each audience, with
expiration based on the tokens' expiration times.  In contrast, gRPC
will essentially have a two-level cache here: the filter will maintain a
cache of GcpServiceAccountIdentityCallCredentials instances for each
audience with expiration based on their respective last-used times,
and each of those GcpServiceAccountIdentityCallCredentials instances
will internally cache the token for its audience based on the token's
expiration time.  In the majority of cases, this is expected to result
in the same behavior, although it is conceivably possible for there to
be edge cases where a given GcpServiceAccountIdentityCallCredentials
instance is retained in the cache due to being used more recently even
though it has actually been failing to obtain a token.  However, this
approach allows for cleaner code and better reuse of existing call
credentials implementations in some languages.

## Implementation

C-core implementation:
- generalize CDS metadata handling (https://github.com/grpc/grpc/pull/37468)
- implement GcpServiceAccountIdentityCredentials
  (https://github.com/grpc/grpc/pull/37544)
- validate Audience cluster metadata (https://github.com/grpc/grpc/pull/37566)
- implement GCP auth filter (https://github.com/grpc/grpc/pull/37550)
- mechanism for retaining cache across xDS updates
  (https://github.com/grpc/grpc/pull/37646)

Will be implemented in all other languages, timelines TBD.
