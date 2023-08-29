A71: xDS Fallback
----
* Author(s): @eostroukhov
* Approver: @markdroth
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-07-27
* Discussion at: https://groups.google.com/g/grpc-io/c/07M6Ua7q4Hc

## Abstract

This proposal describes a fallback mechanism for situtaions when the primary
xDS server is unavailable. xDS support for client-side gRPC is described
in gRFC [A27][A27] and server-side support is described in [A36][A36].

Several specific scenarios need to be considered:
1. The xDS server is not available at the time of the initial connection.
1. The xDS server is no longer available after some resources have been 
obtained.

## Background

xDS (also known as xDS Discovery Service) is a suite of APIs for discovering
and subscribing to the configuration of a server mesh. Even a brief downtime
of the xDS control plane may cause significant disruption in the service mesh
inter-component connectivity and result in wider outages.

Current xDS implementation in gRPC uses a single global instance of
the XdsClient class to communicate with the xDS control plane. This instance
is shared across all channels to reduce overhead by providing a shared cache
for xDS resources as well as reducing the number of connections to xDS servers.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A36: xDS-Enabled Servers][A36]
* [A40: xDS Configuration Dump via Client Status Discovery Service in gRPC][A40]
* [A47: xDS Federation][A47]
* [A57: XdsClient Failure Mode Behavior][A57]

## Proposal

Modify gRPC to use a shared XdsClient instance for each data plane target
instead of using ai single global XdsClient instance. In other words, if there
are two channels created for the target "xds:foo", they will share one
XdsClient instance, but if another channel is created for "xds:bar", it will
use a different XdsClient instance.

This way if some resource for "xds:bar" is not available in cache, only that
XdsClient will switch to a fallback server and refetch the resources or
resubscribe to resource change notifications. "xds:foo" will still be using
cached resources obtained from the primary server.

### Reservations about using the fallback server data

We are expecting to use this in an environment where the data provided by
the secondary xDS server is better than nothing but is less desirable than
the data in the primary server. This means that while we definitely want to use
the fallback data in preference to the client simply failing, we will always
prefer to use the data from the primary server over the data from the fallback
server. In principle, this means that whenever we have a resource already
cached from the primary server, we want to use that instead of falling back.

We have no guarantee that a combination of resources from the primary and
secondary server forms a valid cohesive configuration, so we cannot make this
determination on a per-resource basis. We need any given gRPC channel to either
use all resources from the primary server or all resources from the secondary
server.

The ideal behavior is for each individual gRPC channel to fall back
independently of any other, but the global XdsClient instance makes that
challenging, because we need to make the fallback decision independently for
each authority, and that is handles inside the XdsClient. (And we do still want
to share XdsClient instances across channels, both to reduce load on
the control plane from duplicate channels and to make CSDS work.)

### xDS resources caching

The xDS client code caches any responses received from the xDS server. This
includes:
- Resources that have been successfully received and can be used.
- Resources that were recieved but did not pass the validation.
- Resources that are considered as not existing according
    to [xDS Protocol Specification][resource-does-not-exist].

Resources that have not been recieved by the time xDS server fallback was
initiated are not considered cached.

### Initiating xDS Fallback

The fallback process is initiated if the following conditions hold:

* There had been a failure during resource retrieval, as described in [A57]:
    - Channel reports TRANSIENT_FAILURE.
    - The ADS stream being closed before the first server response had been
        received.
* At least one watcher exists for a resource that is not cached. See above
    for details on resource caching.

### Changes to the code base

1. Refactor code using XdsClient to support multiple instances. 
1. Add support for multiple xDS servers in the bootstrap JSON. Current
    implementation only returns the first entry and ignores the rest.
1. Add multi-client support to CSDS

#### Refactor code using XdsClient to support multiple instances.

1. Introduce a map of "sub-XdsClient" instances keyed by the data plane
    target.
1. For each authority, we need to store an ordered list of channels instead
    of just one channel.
1. Whenever we lose contact with a given server and there is at least one
    resource requested but not cached, if the current server is not the
    last server in the list, then we create a channel/stream for the next
    server in the list.
1. Whenever we get back in contact with a given server, if we have
    a channel/stream for the next server in the list, close it.

#### Add support for multiple xDS servers in the bootstrap JSON

Currently bootstrap JSON supports multiple xDS servers but semantics are not
explicitely specified.

1. xDS servers will be attempted in the order they are specified in
    the bootstrap JSON. Server will only be attempted if the previous entry in
    the list is not available.
1. xDS client will report a failure if the last entry in the list is not
    available.

Currently internal data structures do not allow for more then a single xDS
server. The implementation needs to be updated to handle multiple servers,
maintainig their fallback order.

#### Add multi-client support to CSDS

Ses [A40] for details on CSDS.

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

### Using a single XdsClient but only refetched cached resource if a request was made for a non-cached resource.

This is similar to the option above but causes less refetches. It will still
result in many more fetches than the proposed solution with per-target
XdsClients.

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A40]: A40-csds-support.md
[A47]: A47-xds-federation.md
[A57]: A57-xds-client-failure-mode-behavior.md

[resource-does-not-exist]: https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist
