A66: xDS Fallback
----
* Author(s): @eostroukhov
* Approver: @markdroth
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-07-27
* Discussion at: <google group thread> (filled after thread exists)

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
and subscribing to the configuration of a server mesh. Orignially created by
for the [Envoy Proxy](http://envoyproxy.io) project, the xDS protocol is
evolving into an industry standard. gRPC introduced a number of features that
can be configured using this protocol.

Components of the server mesh communicate with xDS server(s) (also known as
a "control plane") to obtain the initial configuration and to receive updates
to the configuration.

An xDS configuration consists of a number of "resources" that can be queried
separately and may change independently from one another.

Even a brief downtime of the xDS control plane may cause significant disruption
in the service mesh inter-component connectivity and result in wider outages.
Examples include:

* Services becoming unavailable as they fail to access downstream dependencies.
* Traffic being routed incorrectly failing to properly balance the load between
the endpoints.
* Security risks may be introduced as outdated credentials are used for
communications between the servers.

Current xDS implementation in gRPC uses a single global instance of
the XdsClient class to communicate with the xDS control plane. This instance
is shared across all channels to reduce overhead by providing a shared cache
for xDS resources as well as reducing a number of connections to xDS servers.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing](A27)
* [A36: xDS-Enabled Servers](A36)
* [A57: XdsClient Failure Mode Behavior](A57)

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A57]: A57-xds-client-failure-mode-behavior.md

## Proposal

gRPC code should use fallback iff both:

* primary server not reachable
* at least one watcher exists for a resource that is not cached (where
   "does not exist" counts as cached).

Instead of using a global XdsClient instance, gRPC will use a shared XdsClient
instance for each data plane target.  In other words, if two channels are
created for the target "xds:foo", they will share one XdsClient instance, but
if another channel is created for "xds:bar", it will use a different XdsClient
instance.

Attempt to fetch a resource for a cached target will result in a failure.

The following changes will be made to the codebase:

1. Add support for multiple xDS servers in the `bootstrap.json`.
2. Update implemention to support configuration with multiple xDS servers.
3. Refactor XdsClient to support multiple instances.
4. Update code relying on XdsClient to work with multiple instances
    * xDS resource fetching and tracking.
    * xDS stats *[???] (do we want to collect them cross-XdsClients?)*

#### bootstrap.json

Currently `bootstrap.json` supports multiple xDS servers but semantics are
not explicitely specified.

1. xDS servers will be attempted in the order they are specified in
the bootstrap.json. Server will only be attempted if the previous entry in
the list is not available.
1. xDS client will report a failure if the last entry in the list is not
available.
1. `channel-creds` or any other server attributes are not shared and need
to be defined independently for every server.

*[???] Are there any cases when we revert to a primary server?*

#### Internal xDS bootstrap representation

Currently internal data structures do not allow for more then a single xDS
server. The implementation needs to be updated to handle multiple servers,
maintainig their fallback order.

#### XdsClient class changes

Each language implementation will need to ensure that multiple XdsClient
instances may be created and torn down as needed during the application
lifetime.

### Temporary environment variable protection

This option will be behind `GRPC_EXPERIMENTAL_XDS_FALLBACK`. If this variable
is unset or is falsy, only one xDS server will be read from the bootstrap
file. 

## Rationale

Other approaches considered:

#### Always switch to a fallback server if the primary is not available

A single XdsClient is used for all targets. All cached resources are purged and
refetched as soon as primary xDS server becomes unavailable. Major shortcoming
of this approach is that this would put fallback server availability at risk
due to spike in demand once the primary server goes down and all the clients
try to refetch the resources.

#### Always switch to a fallback server if the primary is not available but only if there's a need to fetch the resources that are missing from the cache.

This is similar to an option above but should decrease the load on the fallback
servers.

This may still result in unnecessary refetch in some cases.
