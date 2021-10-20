A47: xDS Federation
----
* Author(s): Mark D. Roth (@markdroth)
* Approver: ejona86, dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-10-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC has the ability to use the xDS APIs for configuration.  Client-side
support is described primarily in gRFCs [A27][A27] and [A28][A28], and
server-side support is described in gRFC [A36][A36].

[xRFC TP1][xRFC TP1] describes a new naming scheme for xDS resources
where resource names are encoded as URIs with an `xdstp` scheme, where
the authority of the URI indicates which xDS server to get the resource
from.  This new naming scheme is the cornerstone to support **federation**,
because it allows xDS clients to get different resources from different xDS
servers without concern about naming conflicts.

This document describes how gRPC will support xDS federation and the new
resource naming scheme.

## Background

This design includes the following features:
- Ability to talk to different xDS servers for different resources,
  keyed by the authority of the resource name.
- Support for new-style xDS resource names, including basic handling of
  context parameters.  (Note: The design has evolved away from using
  context parameters for dynamic resource selection; instead, there is a
  separate dynamic parameter proposal in [xRFC
  TP2](https://github.com/cncf/xds/pull/10) for that.  However, context
  params can still be statically configured in the client's config or
  used in resource names sent by the server.)

This design does *not* include the following features:
- Collection resource types.  This is not something that gRPC currently
  needs, since it does not use collections for LDS or CDS the way that
  Envoy does.
- Incremental xDS protocol support.  This is necessary only to support
  glob collections.  gRPC will eventually need this for scalability
  reasons for [LEDS](https://github.com/envoyproxy/envoy/issues/10373),
  but it's not necessary today, so we are decoupling it in order to
  reduce the scope of this design.
- Support for the `alt` directive.  We do not currently have a use-case
  for which this is required in gRPC.
- xDS redirects.  We do not currently have a use-case for which this is
  required in gRPC.
- Support for dynamic parameters.  This will be necessary in the
  not-too-distance future, but it will be covered in a separate design.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A28: xDS Traffic Splitting and Routing][A28]
* [gRFC A36: xDS-Enabled Servers][A36]
* [xRFC TP1: xdstp:// Structured Resource Naming, Caching, and Federation][xRFC TP1]

[A27]: A27-xds-global-load-balancing.md
[A28]: A28-xds-traffic-splitting-and-routing.md
[A36]: A36-xds-for-servers.md
[xRFC TP1]: https://github.com/cncf/xds/blob/main/proposals/TP1-xds-transport-next.md

## Proposal

This design extends the bootstrap config to encode a list of xDS servers
for each authority.  The existing `xds` URI scheme used in the gRPC
client will be extended to support authorities, and machinery will be
provided to construct new-style xDS resource names for each authority
for both gRPC clients and servers.

### gRPC Client Target URI Syntax

We will continue to support the existing `xds` target URI scheme, since
we expect this to be more convenient for users than having to specify a
full `xdstp` URI (which includes the resource type and the full path to
the resource).  However, we do not want this convenience to come at the
cost of flexibility for the deployment -- i.e., we want the deployment
to be able to decide to use new-style resource names without requiring
users to explicitly specify them.  To that end, we will add support for
an optional authority in the `xds` URI.  The procedure for handling an
`xds` URI is as follows:
- If the `xds` URI contains an authority, we will look up the entry for
  that authority in the bootstrap file's `authorities` map to determine
  which configuration to use.  The
  `client_listener_resource_name_template` field in that entry will be
  used to determine the actual resource name to use.
- If the `xds` URI does *not* contain an authority, we will use the
  bootstrap file's top-level
  `client_default_listener_resource_name_template` field to determine the
  resource name.
  - If the resulting resource name starts with `xdstp:`,
    then we will look up the entry for that authority in the bootstrap
    file's `authorities` map to determine which configuration to use (but
    the `client_listener_resource_name_template` field in that entry will
    not be used).
  - If the resulting resource name does *not* start with `xdstp:`, we will not
    do any lookup in the `authorities` map; instead, we will use the default
    list of xDS servers from the top-level `xds_servers` field.

Note that the authority used for the data plane connections (which is
also used to select the VirtualHost within the xDS RouteConfiguration)
will continue to be the last path component of the `xds` URI used to
create the gRPC channel (i.e., the part following the last `/` character, or
the entire path if the path contains no `/` character).

### Bootstrap Config Changes

This design adds two new fields to the gRPC xDS bootstrap config format,
`authorities` and `client_default_listener_resource_name_template`.  It also
augments the behavior of the existing `server_listener_resource_name_template`
field that was added in [gRFC A36][A36].

The new `authorities` field will have the following format:

```json
// A map of authority name to corresponding configuration.
//
// This is used in the following cases:
// - A gRPC client channel is created using an "xds:" URI that includes
//   an authority.
// - A gRPC client channel is created using an "xds:" URI with no
//   authority, but the "client_default_listener_resource_name_template"
//   field turns it into an "xdstp:" URI.
// - A gRPC server is created and the
//   "server_listener_resource_name_template" field is an "xdstp:" URI.
//
// In any of those cases, it is an error if the specified authority is
// not present in this map.
"authorities": {
  // Entries are keyed by authority name.
  // Note: If a new-style resource name has no authority, we will use
  // the empty string here as the key.
  "<authority_name>": {
    // A template for the name of the Listener resource to subscribe
    // to for a gRPC client channel.  Used only when the channel is
    // created using an "xds:" URI with this authority name.
    //
    // The token "%s", if present in this string, will be replaced
    // with %-encoded service authority (i.e., the path part of the target
    // URI used to create the gRPC channel).
    //
    // Must start with "xdstp://<authority_name>/".  If it does not,
    // that is considered a bootstrap file parsing error.
    //
    // If not present in the bootstrap file, defaults to
    // "xdstp://<authority_name>/envoy.config.listener.v3.Listener/%s".
    "client_listener_resource_name_template": <string>,

    // Ordered list of xDS servers to contact for this authority.
    // Format is exactly the same as the top level "xds_servers" field.
    //
    // If the same server is listed in multiple authorities, the
    // entries will be de-duped (i.e., resources for both authorities
    // will be fetched on the same ADS stream).
    //
    // If not specified, the top-level server list is used.
    "xds_servers": [ ... ]
  }
}
```

The new `client_default_listener_resource_name_template` field will have
the following format:

```json
// A template for the name of the Listener resource to subscribe to
// for a gRPC client channel.  Used only when the channel is created
// with an "xds:" URI with no authority.
//
// If starts with "xdstp:", will be interpreted as a new-style name,
// in which case the authority of the URI will be used to select the
// relevant configuration in the "authorities" map.
//
// The token "%s", if present in this string, will be replaced with
// the service authority (i.e., the path part of the target URI
// used to create the gRPC channel).  If the template starts with
// "xdstp:", the replaced string will be %-encoded.
//
// Defaults to "%s".
"client_default_listener_resource_name_template": <string>,
```

There will be two changes to the existing
`server_listener_resource_name_template` described in [gRFC A36][A36]:
- If the template starts with `xdstp:`, it will be interpreted as a
  new-style name, in which case the authority of the URI will be used to
  select the relevant configuration in the "authorities" map.
- When replacing the `%s` token, if the template starts with `xdstp:`,
  the replacement string will be %-encoded in the resulting resource
  name.

The resulting field will have the following format:

```json
// A template for the name of the Listener resource to subscribe to
// for a gRPC server.
//
// If starts with "xdstp:", will be interpreted as a new-style name,
// in which case the authority of the URI will be used to select the
// relevant configuration in the "authorities" map.
//
// The token "%s", if present in this string, will be replaced with
// the IP and port on which the server is listening.  If the template
// starts with "xdstp:", the replaced string will be %-encoded.
//
// There is no default; if unset, xDS-based server creation fails.
//
// Example:
// "xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/server/%s"
"server_listener_resource_name_template": <string>,
```

#### Example: Bootstrap Config Contains No New Fields

If the bootstrap config does not contain any of the new fields described
above, then here's how the resource names will be determined.

If a gRPC client channel is created for `xds:server.example.com`:
- The target URI has no authority, so gRPC will look at the bootstrap
  config's `client_default_listener_resource_name_template` field, which
  is not set, so it defaults to `%s`.  So the Listener resource name will be
  `server.example.com`, just like it would be prior to this design.
- The Listener resource name does not start with `xdstp:`, so we do not
  use the `authorities` map at all.
- We use the top-level `xds_server` list, which is the same one that we
  would have used prior to this design.

If a gRPC client channel is created for `xds://xds.example.com/server.example.com`:
- The `authorities` map has no entry for `xds.example.com`, so we
  handle this as an invalid URI, which is exactly the same way it
  would have been handled prior to this design.

If a gRPC server is created listening on `0.0.0.0:8080`:
- The existing bootstrap config must already set the server listener
  resource name template to some non-`xdstp` value, so we'll continue
  to use that as the resource name.
- The resource name from template does not start with `xdstp:`, so we
  do not do any lookup in the `authorities` map.
- We default to the top-level `xds_server` list, which is the same one
  that we would have used prior to this design.

#### Example: New-Style Names on gRPC Client

The following new fields are added to the bootstrap config:

```json
"client_default_listener_resource_name_template": "xdstp://xds.example.com/envoy.config.listener.v3.Listener/%s",
"authorities": {
  "xds.example.com": {
  }
}
```

If a gRPC client channel is created for `xds:server.example.com`:
- The target URI specifies no authority, so we use the
  `client_default_listener_resource_name_template` field.  The
  resulting resource name will be
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/server.example.com`.
- The resource name specifies the `xds.example.com` authority, so that's
  the entry we use from the `authorities` map.
- Note: We do not use the `client_listener_resource_name_template`
  field in the `authorities` entry, since we've already used the
  top-level `client_default_listener_resource_name_template` field to
  construct the resource name.
- The `xds_server` list is not specified in the entry for the authority,
  so we default to the top-level `xds_server` list, which is the same one
  that we would have used prior to this design.

If a gRPC client channel is created for `xds://xds.example.com/server.example.com`:
- The target URI specifies the authority `xds.example.com`, so that's the
  entry we use from the `authorities` map.  That entry does not specify any
  `client_listener_resource_name_template` field, so we use the default of
  `xdstp://<authority_name>/envoy.config.listener.v3.Listener/%s`.
  Thus, the resource name will be
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/server.example.com`.
- The `xds_servers` list is not specified in the entry for the authority, so
  we default to the top-level `xds_servers` list, which is the same one that
  we would have used prior to this design.

#### Example: New-Style Names on gRPC Server

The following new fields are added to the bootstrap config:

```json
// Use new-style Listener name on gRPC server.
"server_listener_resource_name_template": "xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/server/%s",

// Authorities map.
"authorities": {
  "xds.example.com": {
  }
}
```

If a gRPC server is created listening on `0.0.0.0:8080`:
- The server Listener resource name will be
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/server/0.0.0.0:8080`.
- The resource name from the template starts with `xdstp:` and has
  authority `xds.example.com`, so that's the entry we look up in the
  `authorities` map.
- The `xds_servers` list is not specified in the entry for the authority, so
  we default to the top-level `xds_servers` list, which is the same one that
  we would have used prior to this design.

#### Example: Multiple Authorities

Consider the following bootstrap config:

```json
// Top-level xDS server list.  Used for all resources with no authority, or
// for any authority that does not specify a list of servers.
"xds_servers": [
  {
    "server_uri": "xds-server.example.com",
    "channel_creds": [ { "type": "google_default" } ]
  }
],

// Resource name template for xds: target URIs with no authority.
"client_default_listener_resource_name_template": "xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/%s?project_id=1234",

// Resource name template for xDS-enabled gRPC servers.
"server_listener_resource_name_template": "xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/server/%s?project_id=1234",

// Authorities map.
"authorities": {
  "xds.example.com": {
    "client_listener_resource_name_template": "xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/%s?project_id=1234"
    }
  },

  "xds.other.com": {
    "xds_servers": [
      {
        "server_uri": "xds-server.other.com",
        "channel_creds": [ { "type": "google_default" } ]
      }
    ]
  }
}
```

If a gRPC client channel is created for `xds:server.example.com`:
- The target URI does not specify any authority, so we use
  `client_default_listener_resource_name_template`, which is
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/%s?project_id=1234`,
  so the resource name will be
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/server.example.com?project_id=1234`.
- We look up authority `xds.example.com` in the `authorities` map.
- Note: We do not use the `client_listener_resource_name_template` field,
  since we have already used the top-level
  `client_default_listener_resource_name_template` field to determine
  the resource name.
- The `xds_servers` list is not specified in the entry for the authority, so
  we default to the top-level `xds_servers` list (pointing to
  `xds-server.example.com`), which is the same one that we would have used
  prior to this design.

If a gRPC client channel is created for `xds://xds.example.com/server.example.com`:
- The target URI specifies authority `xds.example.com`, so we look up
  that entry in the `authorities` map.
- The authority entry's `client_listener_resource_name_template` field is set to
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/%s?project_id=1234`,
  so we use
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/client/server.example.com?project_id=1234`
  as the resource name.
- The `xds_servers` list is not specified in the entry for the authority, so
  we default to the top-level `xds_servers` list (pointing to
  `xds-server.example.com`), which is the same one that we would have used
  prior to this design.

If a gRPC client channel is created for `xds://xds.other.com/server.other.com`:
- The target URI specifies authority `xds.other.com`, so we look up
  that entry in the `authorities` map.
- The authority entry's `client_listener_resource_name_template` field
  is unset but defaults to
  `xdstp://xds.other.com/envoy.config.listener.v3.Listener/grpc/client/%s`,
  so we use
  `xdstp://xds.other.com/envoy.config.listener.v3.Listener/grpc/client/server.other.com`
  as the resource name.
- The `xds_servers` list is specified in the entry for the authority, so
  we use that (pointing to `xds-server.other.com`).

If a gRPC server is created listening on `0.0.0.0:8080`:
- The server Listener resource name will be
  `xdstp://xds.example.com/envoy.config.listener.v3.Listener/grpc/server/0.0.0.0:8080?project_id=1234`.
- The resource name from the template starts with `xdstp:` and has
  authority `xds.example.com`, so that's the entry we look up in the
  `authorities` map.
- The `xds_servers` list is not specified in the entry for the authority, so
  we default to the top-level `xds_servers` list, which is the same one that
  we would have used prior to this design.

### `xds` Resolver Changes

The xds resolver will be responsible for using the bootstrap information
above to determine what Listener resource name to subscribe to for a
given target URI.  The logic will be (pseudocode):

```
target_hostname = StripLeadingSlash(target_uri.path)
if target_uri.authority is set:
  authority_config = bootstrap.LookupAuthority(target_uri.authority)
  if authority_config not found:
    handle as invalid target URI
  else:
    template = authority_config.client_listener_resource_name_template
    if template not set:
      template = ("xdstp://" + target_uri.authority +
                  "/envoy.config.listener.v3.Listener/%s")
    xds_resource_name = ExpandPercentS(template,
                                       PercentEncode(target_hostname))
else:  # target_uri.authority not set
  template = bootstrap.client_default_listener_resource_name_template
  if template not set:
    template = "%s"
  if template starts with "xdstp:":
    target_hostname = PercentEncode(target_hostname)
  xds_resource_name = ExpandPercentS(template, target_hostname)
```

### xDS-Enabled gRPC Server Changes

Similarly to the xds resolver on the client side, the xDS-enabled gRPC
server will need to be changed such that if the
`server_listener_resource_name_template` field in the bootstrap file
starts with `xdstp:`, then it will percent-encode the listening address
before replacing the `%s` token in the template.

### `XdsClient` Changes

This design will require a number of changes in `XdsClient`.

#### Server Channel Map

Currently, the `XdsClient` contains a single server channel.  This will be
replaced with a map of server channels, keyed by the server definition
from the bootstrap file (i.e., target URI, channel creds, and all known
server features -- unknown features can be removed).  This key is used
to de-dup servers, so that if multiple authorities use the same server,
we do not create duplicate channels.

We don't want to proactively connect a channel that is not actually
being used.  Implementations can either proactively create the channel
up-front and depend on idleness behavior, or they can lazily create the
channel when it's actually needed.  Similarly, if a channel stops being
used, implementations can either depend on idleness behavior to tear
down the connections, or they can ref-count the channel and shut it down
when it's no longer needed.  (The expected tear-down after being idle is
O(5-15m).)

#### Resource Cache

The resource cache in the `XdsClient` is currently a two-level map, keyed
first by resource type and then by resource name.  This will need to be
changed to use a three-level map, keyed first by authority, then by
resource type, and then by resource name.

The top-level map in the cache is keyed by authority.  Resources without
any authority will use the empty string as the key.  The value of this
map will include a reference/pointer (details are implementation-dependent)
to the entry in the server channel map for the associated server.  This
indirection supports two things:
- It allows multiple authorities to use the same server channel, via the
  de-duplication described above.
- It will pave the way for future work where each authority can specify
  a list of servers, ordered by priority.  (In other words, the
  authority entry in the cache will change which server channel it is
  using based on which server is currently reachable.)

The second-level map will be the resource type, exactly the same as the
top-level map is today.

The third-level map will be the resource name.  For old-style resource
names, this will be the full resource name, exactly as it is today.  For
new-style names, for optimization purposes, it may be desirable to not
include the authority and resource name in the key here, because that
will duplicate information that is already present from the upper levels
of the map.  One option is to encode it as the `xdstp:` URI minus the
authority and resource type, although the `xdstp:` prefix will still be
needed to indicate that it's a new-style name.  (Implementations could also
do this with a bool instead of the `xdstp:` prefix.)

For example, a resource with old-style name `server.example.com` and
type Listener would be stored like this:

```
{authority "" =>
  {resource_type Listener =>
    {"server.example.com" => <resource>}
  }
}
```

But a resource with new-style name
`xdstp://xds.example.com/envoy.config.listener.v3.Listener/server.example.com`
would be stored like this:

```
{authority "xds.example.com" =>
  {resource_type Listener =>
    {"xdstp:server.example.com" => <resource>}
  }
}
```

New-style resource names used as cache keys must be canonicalized.  This
means sorting the context params in normal lexicographic order.

If a new-style name has duplicate values for the same context param, the
behavior is undefined.  The easiest thing is probably to just use the
last value seen, since this allows the implementation to just populate
the map by iterating over the query params.

#### Watcher Changes

When a caller starts a watch on an old-style resource name (one that
does not start with `xdstp:`), the `XdsClient` will behave essentially as
it does today, using the empty string as the authority name in the
resource cache.

When a caller starts a watch on a new-style resource name (one that
starts with `xdstp:`), the `XdsClient` will parse the URI.  It will use
the authority of the URI as the primary key in the resource cache (using
the empty string if the URI has no authority).  It will canonify any
context parameters that are part of the resource name (i.e., removing
duplicate keys and sorting keys in lexicographic order).  It will then
check the cache to see if the resource is already present, in which case
it will be returned immediately.  Otherwise, it will send a message
subscribing to that resource, using the appropriate xDS server from the
authority.

### LRS Server Representation

We currently represent the LRS server name in several places:
- In the `CdsUpdate` struct returned by the `XdsClient` CDS watcher.
- In the LB policy configs for the `xds_cluster_resolver` and
  `xds_cluster_impl` LB policies.
- In the `XdsClient` load reporting APIs.

Currently, the name is represented as just a string, and it basically
has only two values: unset, meaning that load reporting is not enabled,
or the empty string, meaning to use the same server as we're using for
ADS.  The latter meaning worked fine when there was only one possible
server for us to talk to, but in the presence of federation, there can
be multiple servers, and we need a way to indicate which one we mean.

In the general case, we actually need more than just the server name for
the LRS server; we also need to know what channel creds to use to
contact it.  This is the same information we need in the bootstrap file
for contacting the xDS server itself, so we'll use the same format in
the LB policy configs.  This will allow us to use the same code for
parsing the server specification in both the bootstrap file and the LB
policy configs.

In effect, the object that we use to specify a server in the bootstrap
file will become gRPC's equivalent of the xDS `ConfigSource` message.  In
protobuf form (the "authoritative" representation for the LB policy
configs, even if the code actually uses the JSON form), the message will
be:

```proto
message XdsServer {
  string server_uri = 1;
  string channel_creds_type = 2;
  google.protobuf.Struct channel_creds_config = 3;
  repeated string server_features = 4;
}
```

In C-core, the representation will probably look something like this:

```
struct XdsServer {
  std::string server_uri;
  std::string channel_creds_type;
  Json channel_creds_config;
  std::set<std::string> server_features;

  bool ShouldUseV3() const;

  static absl::StatusOr<XdsServer> Parse(Json json);
};
```

#### `XdsClient` CDS Watcher

For now, the `XdsClient` will continue to require the CDS resource's
`lrs_server` field to contain a `SelfConfigSource` if it is set.  However,
in the future, we could add support for an `ApiConfigSource` specifying a
gRPC server and the channel creds to use (accepting only the same types
of credentials that we grok from the bootstrap file).

In the `XdsClient` API, the `CdsUpdate` struct will need to be changed to
use an `XdsServer` struct instead of a simple string for the LRS server.
The `XdsServer` struct will be populated with the name and channel creds
of the xDS server from which the client obtained the CDS resource (i.e.,
the `SelfConfigSource` will be expanded to an actual name, which can be
passed down to the LB policies).  Note that this means that in the
future, when we add support for xDS server fail-over, if we fail over to
a different xDS server for a CDS resource containing a `SelfConfigSource`
for the LRS field, the `XdsClient` will need to send a new update for the
CDS resource to update the LRS server, even though the actual underlying
resource did not change.

The CDS LB policy will need to use this new field to populate the LB
policy config for the `xds_cluster_resolver` LB policy.

#### LB Policy Config Changes

The configs for the `xds_cluster_resolver` and `xds_cluster_impl` LB
policies will change to use type `XdsServer` instead of type
`google.protobuf.StringValue` for the `lrs_load_reporting_server_name`
field.

The same `XdsServer::Parse()` method will be used in both the bootstrap
file parser and the LB policy config parser in the `xds_cluster_resolver`
and `xds_cluster_impl LB policies`.

#### `XdsClient` Load Reporting APIs

The load reporting APIs will need to accept the LRS server as an
`XdsServer` struct instead of as a simple string.

The load reporting APIs need to store data on a per-LRS-server basis.
(At least in C-core, we are currently passing the LRS server name into
the load reporting APIs, but we're not actually using it for anything
internally.)  Note that in the future, if/when we implement xDS server
failover, this means that when we fail over to a new server, we will
wind up dropping a bit of data that was queued up for the old server,
but that's probably fine.

Note that we currently do not want to trust the control plane to tell us
to contact any arbitrary server with GoogleDefaultCredentials, since
this could leak the oauth token.  We also don't want that attack to
happen via a service config distributed via a DNS TXT record that
configures the `xds_cluster_impl` LB policy to send load reports to some
arbitrary server with GoogleDefaultCredentials.  To avoid this, the
`XdsClient` load reporting APIs will require that the LRS server passed
into them is one of the servers that is already configured in the
bootstrap file (i.e., it matches server name, channel creds, and server
features).

### Temporary environment variable protection

While this feature is in development (until it has passed interop
testing and is deemed production-ready), parsing of the new bootstrap
config fields will be disabled unless the
`GRPC_EXPERIMENTAL_XDS_FEDERATION` environment variable is set to true.

## Rationale

For security reasons, we are focusing on statically configuring a clien
to speak to a specific set of authorities in the bootstrap file rather
than relying on one control plane dynamically telling the client to talk
to a server that it has not before heard of.  This is to avoid leaking
call credentials due to a compromised control plane.

## Implementation

There are a few main steps to the implementation:
- Change `XdsClient` to support multiple authorities.
- Add support for new bootstrap fields and use them in xds resolver and
  xDS-enabled gRPC server.
- Use new LRS server representation in `XdsClient` load reporting APIs and
  LB policy configs.

This is currently being implemented in C-core, Java, and Go.

## Open issues (if applicable)

N/A
