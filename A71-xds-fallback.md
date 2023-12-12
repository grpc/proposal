A71: xDS Fallback
----
* Author(s): @eostroukhov
* Approver: @markdroth
* Status: Draft
* Implemented in: <language, ...>
* Last updated: 2023-11-16
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

Current xDS implementation in gRPC uses a single global instance of
the XdsClient class to communicate with the xDS control plane. This instance
is shared across all channels to reduce overhead by providing a shared cache
for xDS resources and reducing the number of connections to xDS servers.

### Reservations about using the fallback server data

We are expecting to use this in an environment where the data provided by
the secondary xDS server is better than nothing but is less desirable than
the data in the primary server. This means that while we definitely want to use
the fallback data in preference to the client simply failing, we will always
prefer to use the data from the primary server over the data from the fallback
server. In principle, this means that whenever we have a resource already
cached from the primary server, we want to use that instead of falling back.

We have no guarantee that a combination of resources from different xDS servers
form a valid cohesive configuration, so we cannot make this determination on
a per-resource basis. We need any given gRPC channel to use the resources
the same server

### xDS resources caching

The xDS client code caches any responses received from the xDS server. This
includes:

- Resources that have been successfully received and can be used.
- Resources that are considered non-existent according
to [xDS Protocol Specification][resource-does-not-exist].

Note that only attempts at reading a resource that is not cached will trigger
the fallback.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A36: xDS-Enabled Servers][A36]
* [A40: xDS Configuration Dump via Client Status Discovery Service in gRPC][A40]
* [A47: xDS Federation][A47]
* [A57: XdsClient Failure Mode Behavior][A57]

## Proposal

Perform the following changes:

1. Change global XdsClient from a single instance to a map of instances keyed
by channel target.
1. Change xDS bootstrap code to return the full list of servers for each
xDS authority from the bootstrap config.
1. Change XdsClient to support fallback within each xDS authority.

### Change global XdsClient from a single instance to per-target instances

Current gRPC implementation is using a single XdsClient for all data plane
targets. For fallback, this would mean that all the channels would switch to
using the data from the fallback xDS control plane which is not desirable (see
below).

Changes XdsClient to per-data plane target would enable gRPC to switch to
fallback configuration only for channels that need xDS resource that have not
been downloaded yet.

This change would change the channels to get XdsClient for their data plane
target. It will also require changes to the CSDS service to aggregate configs
from multiple XdsClients. For gRPC Core, this would entail updating
`grpc_dump_xds_configs` function to iterate over XdsClients and aggregate the
data available in each. A new field will be added to the CSDS response
to indicate with channel target the data is associated with. See [A40] for
details on CSDS.

gRPC servers using the xDS configuration will share the same XdsClient instance
keyd with a dedicated well-known key value.

### Change xDS Bootstrap to handle multiple xDS configuration servers

Current gRPC xDS implementations only use the first xDS configuration server
listed in the bootstrap JSON document. Fallback implementation requires changes
to use server listed and keep track of their ordering as it defines the
relative priority of the different xDS config servers.

### Change XdsClient to support fallback within each xDS authority

The fallback process is initiated if both of the following conditions hold:

* There is a connectivity failure on the ADS stream, as described in [A57]:
    - Channel reports TRANSIENT_FAILURE.
    - The ADS stream was closed before the first server response had been
    received.
* At least one watcher exists for a resource that is not cached.

This may happen either when setting up a new channel or if the xDS control
plane becomes unavailable after a resource changed notification was received.

XdsClients will need to be changed to support multiple ADS connections for each
authority. Once the fallback process begins, impacted XdsClient will establish
a connection to the next xDS control plane listed in the bootstrap JSON. Then
then XdsClient will subscribed to all watched resources on that server and will
update the cache based on the received responses.

Connecting to the lower-priority servers does not close gRPC connections to the
higher-priority servers. XdsClient will still wait for xDS resources on the ADS
stream. Once such resource is received, XdsClient will close connections to the
lower-priority server(s) and will use the resources from the higher priority
servers.

### Potential issues

#### Config tears

Config tears happen when the client winds up using some combination
of resources from the primary and fallback servers at the same time, even
though that combination of resources was never validated to work together. In
theory, this can cause correctness issues where we might send traffic to
the wrong location or in the wrong way, or it can cause RPCs to fail. Note
that this can happen only when the primary and fallback server use the same
resource names.

This may also happen in non-fallback cases with the configs that span several
xDS resources. The main case where this is likely to come up in practice is
HTTP filter configs split across LDS and RDS. Control planes may avoid that
problem by inlining the RouteConfiguration into the LDS resource.

gRPC team considered addressing this in the fallback case by making the source
xDS server name to the XdsClient but ultimately decided against it as it would
require a significant implementation work and would not recieve sufficient
testing and may reduce reliability instead of increasing it. It would still not
help with config tears in non-fallback cases.

#### Flapping

Flapping is when the client switches over to a different server for LDS and
RDS but then the server dies before it can send CDS, so the client switches
back to the previous server and then re-updates LDS and RDS. This would cause
the client to fail RPCs for a short period of time. Note that this can happen
only when the primary and fallback server use different resource names.

This will be addressed by moving the CDS and EDS watchers to the xDS resolver.

## Other approaches considered:

### Uncoditionally perform fallback when xDS control plane becomes unreachable

In this scenario, XdsClients discard cached resources and fetch new resources
from the fallback server as soon as xDS server becomes unreachable.

Pros:
- No config tears
- Straightforward to implement
- Compatible with Envoy

Cons:
- Increases load on xDS servers
- Decreases data quality by using data from the fallback server.

### Do not split the XdsClient per-target and perform fallback when the required resource is not in the cache.

Pros:
- Avoids fallback in most cases
- Straightforward to implement
- No config tearing
- Compatible with Envoy
Cons:
- May still unnecessarily discard better config from primary server
    - New channel to a new target
    - Primary server goes down while client is processing an update

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
