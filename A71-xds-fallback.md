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

Several specific scenarios need to be considered:
1. The xDS server is not available at the time of the initial connection.
1. The xDS server is no longer available after some resources have been 
obtained.
1. xDS server becomes available after the switch to the fallback server has
been performed.

## Background

xDS (also known as xDS Discovery Service) is a suite of APIs for discovering
and subscribing to the configuration of a server mesh. Even a brief downtime
of the xDS control plane may cause significant disruption in the service mesh
inter-component connectivity and cause wider outages.

Current xDS implementation in gRPC uses a single global instance of
the XdsClient class to communicate with the xDS control plane. This instance
is shared across all channels to reduce overhead by providing a shared cache
for xDS resources and reducing the number of connections to xDS servers.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A36: xDS-Enabled Servers][A36]
* [A40: xDS Configuration Dump via Client Status Discovery Service in gRPC][A40]
* [A47: xDS Federation][A47]
* [A57: XdsClient Failure Mode Behavior][A57]

## Proposal

The fallback process is initiated if both of the following conditions hold:

* There had been a failure during resource retrieval, as described in [A57]:
    - Channel reports TRANSIENT_FAILURE.
    - The ADS stream was closed before the first server response had been
    received.
* At least one watcher exists for a resource that is not cached. See above
for details on resource caching.

This may happen either when setting up a new channel or if the xDS control
plane becomes unavailable after a resource changed notification was received.

Bootstrap configuration will be extended to add support for multiple
configuration servers. Fallback process will attempt to connect
to configuration servers in the order they are listed in the bootstrap
configuration file.

gRPC will switch back to a higher priority configuration server as soon as
the connection to said server is reestablished.

### Introduce per-target XdsClients

In current implementation, all channels use a single XdsClient instance
to obtain the resources. This allows caching the resources and reducing
the load on the xDS servers. Introducing fallback support in this architecture
would mean that we will have to reload configuration for all channels, possibly
interrupting the ones that are already fully configured and servicing
the gRPC traffic.

On the other side of the spectrum is having an XdsClient per channel. This will
reduce opportunities for caching the data, increasing the memory consumption
and load on the xDS servers. This would also increase the overhead
of establishing new channels even when the xDS control plane is fully stable.

Proposed solution is a middle ground. Here, we have a single shared XdsClient
channel per data plane target. gRPC channels to the same target will be using
the same control plane data and will perform fallback at the same time.
Channels that were fully configured by the point the fallback is needed will
keep using the cached data and no updates will be sent to those channels.

In other words, if there are two channels created for the target "xds:foo",
they will share one XdsClient instance, but if another channel is created
for "xds:bar", it will use a different XdsClient instance. This way if some
resource for "xds:bar" is not available in cache, only that XdsClient will
switch to a fallback server and refetch the resources or resubscribe to resource
change notifications. "xds:foo" will still be using cached resources obtained
from the primary server.

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

### Switching back to a higher-priority server

xDS client will switch to the higher-priority server as soon as following
conditions are met:

1. ADS connection to the server is (re)-established.
2. A resource is downloaded from the server.

Note that unlike the fallback sequence, switching back can happen even if all
the resources are cached. See above section for motivation behind this behavior.

### xDS resources caching

The xDS client code caches any responses received from the xDS server. This
includes:

- Resources that have been successfully received and can be used.
- Resources that are considered non-existent according
to [xDS Protocol Specification][resource-does-not-exist].

Note that only attempts at reading a resource that is not cached will trigger
the fallback.

## Changes to the code base

1. Add multi-client support to CSDS
1. Refactor code using XdsClient to support multiple instances. 
1. Add support for multiple xDS servers in the bootstrap JSON. Current
implementation only returns the first entry and ignores the rest. This should
include the changes within the XdsClient to support fallback within a given
authority.


#### Refactor code using XdsClient to support multiple instances.

Instead of having a single XdsClient, introduce an XdsClient instance
per target. 

#### Add support for multiple xDS servers in the bootstrap JSON

1. For each target, we need to store an ordered list of channels instead of
just one channel.
1. Whenever we lose contact with a given server and there is at least
one resource requested but not cached or have previously lost contact and
an uncached resource is requested, we create a channel/stream for the next
server in the list unless we are at the last one.
1. Whenever we get back in contact with a given server, if we have
a channel/stream for a server later in the list, close it.

Currently bootstrap JSON supports multiple xDS servers but semantics are not
explicitly specified.

1. xDS servers will be attempted in the order they are specified in
the bootstrap JSON. Server will only be attempted if the previous entry in
the list is not available.
1. xDS client will report a failure if the last entry in the list is
not available.

In some gRPC implementations, the internal data structures do not allow for more
than a single xDS server. The implementation needs to be updated to handle
multiple servers, maintaining their fallback order.

#### Add multi-client support to CSDS

See [A40] for details on CSDS.

Add add a field to the ClientConfig message in the CSDS response to indicate
which channel target the data is associated with.

### Temporary environment variable protection

This option will be behind `GRPC_EXPERIMENTAL_XDS_FALLBACK`. If this variable
is unset or is false, only one xDS server will be read from the bootstrap
file. 

## Rationale

Other approaches considered:

### Using a single XdsClient instance for all data plane targets

Using a singleton XdsClient instance would cause individual channels that
already have all of their resources cached from the primary server to switch
to fallback resources in the following cases:

- If the primary server goes down and a gRPC client then creates a new
    xDS-enabled channel for a target for which the resources are not already
    cached, that will trigger the client to fall back, not just for the new
    channel but for all channels.
- Cases where the primary server goes down while the client is in the middle
    of processing an update (referred to as the "mid-update" case).
    For example, let's say that the XdsClient gets an updated RouteConfig that
    includes a new cluster that it was not previously using, thus triggering
    the client to subscribe to the new Cluster resource, but then TD goes down
    before sending the new Cluster resource. At this point, the XdsClient will
    fall back to the secondary server, and it will use the fallback resources
    instead of the cached resources that it already had from the primary server.

The best compromise that we can find is to use a different XdsClient instance
for each target, but still share those XdsClient instances between channels for
the same target. This is not exactly the same as having each individual channel
fall back independently, but it's close enough in practice: if there are
multiple channels for the same target, they will all use the same decision
about whether or not to fall back, but that should be fine, since they are also
using the same set of xDS resources.

### Using a single XdsClient but only refetch cached resources if a request was made for a non-cached resource.

This is similar to the option above but also reduce a number of requests to
the control plane in case of fallback. This option still results in many
more fetches than the proposed solution with per-target XdsClients.

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A40]: A40-csds-support.md
[A47]: A47-xds-federation.md
[A57]: A57-xds-client-failure-mode-behavior.md

[resource-does-not-exist]: https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist
