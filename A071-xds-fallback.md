A71: xDS Fallback
----
* Author(s): @eostroukhov
* Approver: @markdroth
* Status: Final
* Implemented in: <language, ...>
* Last updated: 2024-01-04
* Discussion at: https://groups.google.com/g/grpc-io/c/07M6Ua7q4Hc

## Abstract

This proposal describes a fallback mechanism for situations when the primary
xDS server is unavailable. xDS support for client-side gRPC is described
in gRFC [A27][A27] and server-side support is described in [A36][A36].

## Background

xDS (also known as xDS Discovery Service) is a suite of APIs for discovering
and subscribing to the configuration of a server mesh. Even a brief downtime
of the xDS control plane may cause significant disruption in the service mesh
inter-component connectivity and cause wider outages.

The current xDS implementation in gRPC uses a single global instance of
the XdsClient class to communicate with the xDS control plane. This instance
is shared across all channels and xDS-enabled gRPC server listeners to reduce
overhead by providing a shared cache for xDS resources and reducing the number
of connections to xDS servers.

### Reservations about using the fallback server data

We are expecting to use this in an environment where the data provided by
the secondary xDS server is better than nothing but is less desirable than
the data in the primary server. E.g. a fallback environment that uses
an outdated static configuration. This means that while we definitely want
to use the fallback data in preference to the client simply failing, we will
always prefer to use the data from the primary server over the data from
the fallback server. In principle, this means that whenever we have a resource
already cached from the primary server, we want to use that instead of falling
back.

We have no guarantee that a combination of resources from different xDS servers
form a valid cohesive configuration, so we cannot make this determination on
a per-resource basis. We need any given gRPC channel or server listener to only
use the resources from a single server.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A36: xDS-Enabled Servers][A36]
* [A40: xDS Configuration Dump via Client Status Discovery Service in gRPC][A40]
* [A47: xDS Federation][A47]
* [A57: XdsClient Failure Mode Behavior][A57]

## Proposal

Implement the following changes

1. Change global XdsClient from a single instance to a map of instances keyed
by channel target.
1. Update CSDS to aggregate configs from multiple XdsClient instances.
1. Change xDS bootstrap code to return the full list of servers for each
xDS authority from the bootstrap config.
1. Change XdsClient to support fallback within each xDS authority.

### Change global XdsClient from a single instance to per-target instances

Current gRPC implementation is using a single XdsClient for all data plane
targets. For fallback, if any one channel triggers fallback, then all channels
regardless of authority would switch to using the data from the fallback xDS
control plane, which is not desirable.

gRPC servers using the xDS configuration will share the same XdsClient instance
keyed with a dedicated well-known key value.

Implementations will change the XdsClient to be per-data plane target to enable
switch to fallback configuration only for channels that need xDS resources that
have not been downloaded yet. This proposal details changing the channels
to get XdsClient for their data plane target.

### Update CSDS to aggregate configs from multiple XdsClient instances.

The CSDS RPC service will be changed to get data from all XdsClient instances.
A new string field named `client_scope` will be added to the CSDS
[ClientConfig] message to indicate the channel target the data is associated
with. For gRPC clients, this field will contain the key channels use to lookup
their XdsClient instance, such as dataplane target for client channels or
special value "#server" for xDS-enabled gRPC servers.

```
// For xDS clients, the scope in which the data is used. 
// For example, gRPC indicates the data plane target or that the data is
// associated with gRPC server(s).
string client_scope = 4;
```

Adding this field will not require updating most CSDS clients as any given
resource will have only a single `client_scope` in most cases.

### Change xDS Bootstrap to handle multiple xDS configuration servers

Current gRPC xDS implementations only use the first xDS configuration server
listed in the bootstrap JSON document. Fallback implementation requires changes
to use all servers listed in order from highest to lowest priority. Priority is
assigned based on server order in bootstrap.json.

### Change XdsClient to support fallback within each xDS authority

The fallback process is initiated if both of the following conditions hold:

* There is a connectivity failure on the ADS stream, as described in [A57] this
means either:
    - Channel reports TRANSIENT_FAILURE.
    - The ADS stream was closed before the first server response had been
    received.
* At least one watcher exists for a resource that is not cached. Cached
resources include:
    - Resources that have been successfully received and can be used.
    - Resources that are considered non-existent according
    to [xDS Protocol Specification][resource-does-not-exist].

This may happen during a new channel setup or mid-config update, such as
when the control plane sends an updated RDS resource that points to a new
cluster, but then the control plane becomes unavailable before the new CDS
resources can be obtained.

XdsClients will need to be changed to support multiple ADS connections for each
authority. Once the fallback process begins, an impacted XdsClient will
establish a connection to the next xDS control plane listed in the bootstrap
JSON. Then XdsClient will subscribe to all watched resources on that server
and will update the cache based on the received responses.

Connecting to the lower-priority servers does not close gRPC connections to the
higher-priority servers. XdsClient will still wait for xDS resources on the ADS
stream. Once such a resource is received, the XdsClient will close connections
to the lower-priority server(s) and will use the resources from the higher
priority servers.

### Potential issues

#### Config tears

Config tears happen when the client winds up using some combination
of resources from the primary and fallback servers at the same time, even
though that combination of resources was never validated to work together. In
theory, this can cause correctness issues where we might send traffic to
the wrong location or the wrong way, or it can cause RPCs to fail. Note
that this can happen only when the primary and fallback server use the same
resource names.

Config tears can also happen in non-fallback cases with configs that span
several xDS resources, and this is not something that the client can address
in that more general case, so this is something control planes already need to
be aware of. We considered trying to address it for the fallback case by adding
the xDS server name to the XdsClient watcher API, so that it became part of
the key for the dependent watchers, thus ensuring that we never mix resources
from the primary or fallback servers. However, this would have required
significant implementation work and would have introduced a seldom-used and
therefore less well-exercised code-path that would likely have reduced
reliability instead of increasing it. And since this would not have helped for
the more general case anyway, we decided not to pursue it.

The main case where config tears is likely to come up in practice is HTTP
filter configs split across LDS and RDS. We recommend that control planes avoid
that problem by inlining the RouteConfiguration into the LDS resource.

#### Flapping

Flapping is when the client switches over to a different server for LDS and
RDS but then the server dies before it can send CDS, so the client switches
back to the previous server and then re-updates LDS and RDS. This would cause
the client to fail RPCs for a short period of time. Note that this can happen
only when the primary and fallback server use different resource names.

This will be addressed in a subsequent gRFC by moving the CDS and EDS watchers
to the xDS resolver.

## Rationale

The following subsections describe various alternatives we considered.

### Perform fallback based only on xDS server reachability

In this scenario, XdsClients discard cached resources and fetch new resources
from the fallback server as soon as xDS server becomes unreachable. This was
rejected because it would cause us to fallback even when everything is cached,
which is undesirable.

Pros:
- No config tears
- Straightforward to implement
- Compatible with Envoy

Cons:
- Increases load on xDS servers
- Decreases data quality by using data from the fallback server.

### Keep a global XdsClient

This was rejected because it would mean that if any one channel had a problem
that triggered fallback, it would trigger fallback for all channels, even those
that already have all of their resources cached.

Pros:
- Avoids fallback in most cases
- Straightforward to implement
- No config tearing
- Compatible with Envoy

Cons:
- May still unnecessarily discard better config from primary server
    - New channel to a new target
    - Primary server goes down while client is processing an update

### Have a separate XdsClient per gRPC channel

Cons:
- Increases xDS traffic when there are multiple channels for the same target.
- Would break CSDS

### Perform fallback at resource granularity

This approach would drastically increase the likelihood of config tears.

### Modify gRPC to support atomic config updates

Builds on top of previous option but also move resource watches from
LB policies to XdsResolver. Redesign XdsClient watch API to express
relationships between resource watches.

## Temporary environment variable protection

This option will be behind `GRPC_EXPERIMENTAL_XDS_FALLBACK`. If this variable
is unset or is false, only one xDS server will be read from the bootstrap
file. 

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A40]: A40-csds-support.md
[A47]: A47-xds-federation.md
[A57]: A57-xds-client-failure-mode-behavior.md

[resource-does-not-exist]: https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist
[ClientConfig]: https://github.com/envoyproxy/envoy/blob/e61e461736a28e26b6fcf0ca25d34c47ed29b0fc/api/envoy/service/status/v3/csds.proto#L130
