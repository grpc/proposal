A27: xDS-Based Global Load Balancing
----
* Author(s): Mark D. Roth
* Approver: a11r
* Status: Draft
* Implemented in: (in progress in C-core, Java, and Go)
* Last updated: 2020-01-07
* Discussion at: https://groups.google.com/d/topic/grpc-io/GWIH4XhIqHA/discussion

## Abstract

gRPC currently supports its own ["grpclb"
protocol](https://github.com/grpc/grpc-proto/blob/master/grpc/lb/v1/load_balancer.proto)
for [look-aside
load-balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md).
However, the popular [Envoy](http://envoyproxy.io) proxy uses the [xDS
API](https://github.com/envoyproxy/data-plane-api/tree/master/envoy/api/v2)
for many types of configuration, including load balancing, and that API
is evolving into a standard that will be used to configure a variety of
data plane software.  In order to converge with this industry trend, gRPC
will be moving from its original grpclb protocol to the new xDS protocol.

The xDS protocol allows configuring a variety of features, including
many that gRPC does not currently support.  Over time, we will be slowly
adding support for more of these features in gRPC.  We will publish a
gRFC for each set of xDS features as we add support for them.

This document describes the changes we are making to support global load
balancing features, which is the set of xDS functionality that we are
initially targetting.  This affects the gRPC client only; use of xDS in
the gRPC server will come later as part of a separate set of xDS features.

## Background

The xDS API is actually a suite of APIs with names of the form "x
Discovery Service", where there are many values for "x" (hence the name
"xDS" for the overall protocol suite).  There is not currently a formal
spec for the protocol, but the best reference is at
https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol.
Readers of this document should familiarize themselves with the xDS API
before proceeding.

The xDS APIs are essentially pub/sub mechanisms, where each API allows
subscribing to a different type of configuration resource.  In the xDS
API flow, the client uses the following main APIs, in this order:

- __Listener Discovery Service (LDS)__: Returns
  [`Listener`](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/listener.proto#L27)
  resources.  Used basically as a convenient root for the gRPC client's
  configuration.  Points to the RouteConfiguration.
- __Route Discovery Service (RDS)__: Returns
  [`RouteConfiguration`](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/route.proto#L24)
  resources.  Provides data used to populate the gRPC [service
  config](https://github.com/grpc/grpc/blob/master/doc/service_config.md).
  Points to the Cluster.
- __Cluster Discovery Service (CDS)__: Returns
  [`Cluster`](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/cluster.proto#L34)
  resources.  Configures things like load balancing policy and load reporting.
  Points to the ClusterLoadAssignment.
- __Endpoint Discovery Service (EDS)__: Returns
  [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/endpoint.proto#L33)
  resources.  Configures the set of endpoints (backend servers) to load
  balance across and may tell the client to drop requests.

gRPC will support the Aggregate Discovery Service (ADS) variant of xDS,
where all of these resource types are obtained on a single gRPC stream.
However, the different phases of the API flow are still typically
referred to using the separate service names described above.

In the future, we may add support for the incremental ADS variant of
xDS.  However, we have no plans to support any non-aggregated variants
of xDS, nor do we plan to support REST or filesystem subscription.

Initially, gRPC will support version 2 of the xDS APIs.  In the future,
we will likely upgrade to v3 or beyond, with appropriate transition
periods to allow users to upgrade their management servers.

### Related Proposals: 

[A24: Load Balancing Policy Configuration](https://github.com/grpc/proposal/blob/master/A24-lb-policy-config.md)

## Proposal

### gRPC Client Architecture

Because xDS handles not only load balancing but also service discovery
and configuration, gRPC will support xDS via both the resolver and LB
policy plugins.  Both the resolver and LB policy plugins will need to
communicate with the xDS server, so the functionality for interacting with
the xDS server will be contained within an XdsClient object, which will
be used by both the resolver and the LB policies.

There will be two separate LB policies for xDS, one to support `Cluster`
data, and another to support `ClusterLoadAssignment` data.

Note that the LB policies will have "experimental" suffixes for their
names for now.  These suffixes will be removed when the functionality
has proven to be stable.

![gRPC Client Architecture Diagram](A27_graphics/grpc_client_architecture.png)

[Link to SVG file](A27_graphics/grpc_client_architecture.svg)

#### XdsClient and Bootstrap File

The XdsClient object contains all of the logic for interacting with the
xDS server.  The XdsClient object is a purely internal API, not
something exposed to gRPC users, but it is described here as part of
the architecture used to support xDS in the gRPC client.

The XdsClient object is configured via a bootstrap file.  The location
of the bootstrap file is determined via the `GRPC_XDS_BOOTSTRAP`
environment variable.  The file is in JSON form, and its contents look
like this:

```
{
  // The xDS server to talk to.  The value is an array to allow for a
  // future change to add support for failing over to a secondary xDS server
  // if the primary is down, but for now, only the first entry in the
  // array will be used.
  xds_servers": [
    {
      "server_uri": <string containing URI of xds server>,
      // List of channel creds; client will stop at the first type it
      // supports.  This field is optional; if not specified, xDS will use
      // the same channel creds as the backends, but with any associated
      // call creds stripped off.
      "channel_creds": [
        {
          "type": <string containing channel cred type>,
          // The "config" field is optional; it may be missing if the
          // credential type does not require config parameters.
          "config": <JSON object containing config for the type>
        }
      ]
    }
  ],
  "node": <JSON form of Node proto>
}
```

Initially, the only type of channel creds we support will be
`google_default`.  In the future, we will add a general-purpose
mechanism for configuring arbitrary channel creds types with arbitrary
configuration.

The `node` field will be populated with the [xDS `Node`
proto](https://github.com/envoyproxy/data-plane-api/blob/1adb5d54abb0e28ca409254d26fad1cf5535239b/envoy/api/v2/core/base.proto#L85).
Note that the `build_version`, `user_agent_name`, `user_agent_version`, and
`client_features` fields should not be specified in the bootstrap file, since
they will be populated automatically by the gRPC xDS client.

To allow for future changes in a backward-compatible way, unknown fields
in the bootstrap file will be silently ignored.

#### xds Resolver

Clients will enable use of xDS by using the `xds` resolver in the
target URI used to create the gRPC channel.  For example, a user may
create a channel using the URI "xds:example.com:123" or
"xds:///example.com:123", which will use xDS to establish contact with
the server "example.com:123".  The "xds" URI scheme does not support any
authority.

When the channel attempts to connect, the `xds` resolver will
instantiate an XdsClient object and use it to send an xDS request for
a `Listener` resource with the name of the server that the client
wishes to connect to (in the example above, "example.com:123").  If the
resulting `Listener` does not include the `RouteConfiguration`,
then the client will also send an xDS request for that resource.
This information will be used to construct the gRPC [service
config](https://github.com/grpc/grpc/blob/master/doc/service_config.md),
which the resolver will return to the channel.

The `xds` resolver will return a service config that selects use of the CDS LB
policy, described below.  The configuration of the CDS policy will indicate
which `Cluster` resource will be used.

Note that the `xds` resolver will return an empty list of addresses, because
in the xDS API flow, the addresses are not returned until the
`ClusterLoadAssignment` resource is obtained later.

The `xds` resolver will also return a reference to the XdsClient object
that it instantiated.  This reference will be passed down to the LB
policies, which is how the resolver and LB policies can share the same
XdsClient object.

#### CDS LB Policy

The CDS LB policy will be the top-level LB policy.  Its configuration
will be of the following form:

```
{
  "cluster": "<cluster name>"
}
```

The CDS policy will use the XdsClient object passed in from the `xds` resolver
to query for the `Cluster` resource with the cluster name from the LB
policy configuration.

Initially, the CDS policy will always create an EDS policy as the child
policy.  In the future, this may change; for example, we may add support
for clusters that use DNS or for aggregate clusters, which would require
different child policies.

#### EDS LB Policy

The EDS LB policy will be the child of the CDS LB policy.  Its
configuration will be of the following form:

```
{
  "edsServiceName": "<EDS service name>",
  // Optional; if not specified, load reporting will be disabled.
  // If value is the empty string, that means to use the xDS server as
  // the LRS server.  (Currently, we do not support non-empty values;
  // support for this will be added later.)
  "lrsLoadReportingServerName": "<LRS server name>"
}
```

The EDS policy will use the XdsClient object passed in from the xds
resolver to query for the `ClusterLoadAssignment` resource with the
EDS service name from the LB policy configuration.

Note that unlike the old grpclb policy, which just received a flat list
of backend servers from the balancer to round-robin across, the EDS policy 
receives a hierarchical list, where the servers are grouped into
localities, and the localities are grouped into priorities.  The EDS
policy will be responsible for both selecting the highest priority set
of localities that are reachable and distributing the traffic across the
localities in the selected priority based on the locality weights.

Note that the EDS policy will *not* support
[overprovisioning](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overprovisioning),
which is different from Envoy.  Envoy takes the
overprovisioning into account in both [locality-weighted load
balancing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight)
and [priority
failover](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority),
but gRPC assumes that the xDS server will update it to redirect traffic
when this kind of graceful failover is needed.  gRPC will send the
[`envoy.lb.does_not_support_overprovisioning` client
feature](https://github.com/envoyproxy/envoy/pull/10136) to the xDS
server to tell the xDS server that it will not perform graceful failover;
xDS server implementations may use this to decide whether to perform
graceful failover themselves.

Unlike the grpclb policy, which relied on server-side load reporting to
the balancer, the EDS policy will perform client-side load reporting
using the [LRS
protocol](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/service/load_stats/v2/lrs.proto).
Note that the EDS policy will not support per-endpoint stats; it will
report only per-locality stats.  Currently, the EDS policy will report
only client-side stats; in the future, we will support also reporting
backend server stats reported to the client via
[ORCA](https://docs.google.com/document/d/1NSnK3346BkBo1JUU3I9I5NYYnaJZQPt8_Z_XCBCI3uA/edit).

### How gRPC Will Use the xDS API

gRPC does not have exactly the same set of features as Envoy, so there
are by necessity some small differences between how the two clients use
the xDS API.  This section documents exactly what gRPC requires from the
xDS server and what xDS features it does and does not support.

Because there are many fields in the xDS APIs that gRPC will not use in
its initial version, the gRPC client must in general ignore any fields
it does not support.  This does mean that if we add support for one of
these fields in a future version and users enable use of the feature in
the management server, older clients will continue to ignore the field,
so they may not actually get the new behavior.  However, the alternative
is even worse: if we make the client strict (i.e., it will fail if the
management server populates fields that the client does not yet support),
then the same scenario will cause the older clients to fail instead of
just ignoring the new feature.

Note, however, that there are some unavoidable cases where enabling a
new feature will cause older clients to break.  These specific cases
will be called out below.

#### Initial Request on the ADS Stream

The first request on the ADS stream will include a Node message that
identifies the client.  As mentioned above, most of the fields for the Node
message will be read from the bootstrap file.

#### LDS

The gRPC client will typically start by sending an LDS request for a
`Listener` resource whose name matches the server name from the target
URI used to create the gRPC channel (including port suffix if present).
Note that the server name will also be used in the HTTP/2 ":authority"
field, so it needs to be in host or host:port syntax.  However, the gRPC
client will not interpret it in any special way.  This means that the
xDS server effectively defines what the default port means if the URI
does not specify a port.

Because we are requesting one specific resource by name, the LDS response
should include at least one `Listener`.  However, many existing xDS servers
do not support requesting one specific resource in LDS, so the gRPC client
must be tolerant of servers that may ignore the requested resource name;
if the server returns multiple resources, the client will look for the one
with the name it asked for, and it will ignore the rest of the entries.
Note that this means that the resource returned by the xDS server must
have exactly the name specified by the client.

#### Listener Proto

The returned `Listener` should be an "HTTP API listener", using the format
added to the xDS API in https://github.com/envoyproxy/envoy/pull/8170.
The HTTP API listener configuration may either provide the
`RouteConfiguration` directly inline, or it may tell the client to
use RDS.  Note that if using RDS, the __ConfigSource__ proto for the RDS
server must indicate to use ADS.

#### RDS

If the `Listener` does not include the `RouteConfiguration` inline,
then the client will send an RDS request asking for the specific
`RouteConfiguration` name specified in the `Listener`.

Because we are requesting one specific resource by name, the RDS response
should include no more than one RouteConfiguration.  However, the gRPC
client must be tolerant of servers that may ignore the resource name in
the request; if the server returns multiple resources, the client will
look for the one with the name it asked for, and it will ignore the rest
of the entries.

#### RouteConfiguration Proto

The `RouteConfiguration` includes a list of zero or more __VirtualHost__
resources.  The gRPC client will look through this list to find an element
whose domains field matches the server name specified in the "xds:" URI.
If no matching VirtualHost is found in the RouteConfiguration, the `xds`
resolver will return an error to the client channel.

In our initial implementation, the only field in the VirtualHost proto
that the gRPC client needs to look at is the list of routes.  The client
will look only at the last route in the list (the default route),
whose match field must contain a prefix field whose value is the empty
string and whose route field must be set.  Inside that route message,
the cluster field will indicate which cluster to send the request to.

If the cluster cannot be determined, the `xds` resolver will return an
error to the client channel.

If the cluster can be determined, the `xds` resolver will return a service
config that selects the CDS LB policy.  The LB policy's configuration will
include the name of the cluster.

Notes on backward compatibility:

- Initially, the gRPC client will look only at the last route in the list,
  which is expected to be the default route.  In the future, we will add
  support for multiple routes, which may be selected based on which RPC
  method is being called or possibly on a header match (details TBD).
- Initially, the gRPC client will require that the route action specify a
  single cluster, as described above.  However, in the future, we will add
  support for the `weighted_clusters` field, which will allow us to support
  traffic splitting.  (This will require introducing a new top-level LB
  policy.)
- The `weighted_clusters` field is in the same "oneof" as the `cluster` field.
  This means that older clients requiring the `cluster` field will
  stop working if the `weighted_clusters` field is specified instead.
  Unfortunately, there does not seem to be a way to avoid this, so it's
  something that our early users will need to be aware of: when we add
  support for `weighted_clusters`, they must upgrade their clients before
  they can start using this feature.  (We might be able to ameliorate
  this by adding support for route matchers at the same time as we
  add `weighted_clusters` support, in which case the final route in the
  list can continue to use the `cluster` field but earlier routes can
  use `weighted_clusters`.)

#### CDS

Once the cluster name is obtained from the __VirtualHost__ proto, the gRPC
client will make a CDS request asking for that specific cluster name.

Because we are requesting one specific resource by name, the CDS response
should include at least one `Cluster`.  However, many existing xDS servers
do not support requesting one specific resource in CDS, so the gRPC client
must be tolerant of servers that may ignore the requested resource name;
if the server returns multiple resources, the client will look for the one
with the name it asked for, and it will ignore the rest of the entries.
Note that this means that the resource returned by the xDS server must
have exactly the name specified by the client.

#### Cluster Proto

The `Cluster` proto must have the following fields set:

- The `type` field must be set to EDS.
- In the `eds_cluster_config` field, the `eds_config` field must be set to
  indicate to use EDS (and the __ConfigSource__ proto for the EDS server
  must indicate to use ADS).  If the `service_name` field is set, that value
  will be used for the EDS request instead of the cluster name.
- The `lb_policy` field must be set to `ROUND_ROBIN`.
- If the `lrs_server` field is set, it must have its `self` field set, in
  which case the client should use LRS for load reporting.  Otherwise
  (the `lrs_server` field is not set), LRS load reporting will be disabled.

#### EDS

After getting the `Cluster` proto, the gRPC client will make an EDS
request for a specific resource name, using either the cluster name or
the value of the `eds_cluster_config.service_name` field in the
`Cluster` proto.

#### ClusterLoadAssignment Proto

The `ClusterLoadAssignment` proto must have the following fields set:

- The `endpoints` field must contain at least one entry.  In each entry:
  - If the `load_balancing_weight` field is unset, the `endpoints` entry
    is skipped; otherwise, the value is used for weighted locality picking.
  - The `priority` field must be set.
  - The `lb_endpoints` field must contain at least one entry.  In each entry:
    - If the `health_status` field has any value other than HEALTHY or
      UNKNOWN, the entry will be ignored.
    - The `endpoint` field must be set.  Inside of it:
      - The `address` field must be set.  Inside of it:
        - The `socket_address` field must be set.  Inside of it:
          - The `address` field must be set to an IPv4 or IPv6 address.
          - The `port_value` field must be set.
- The `policy.drop_overloads` field may be set.
- Note: The `policy.overprovisioning_factor` field will be ignored if
  set; see description of the EDS LB policy above for details.

## Rationale

This design could have been much simpler if we never intended to
configure anything other than load balancing via xDS, because we could
have done all of the xDS integration inside of a single LB policy, much
like we did with grpclb.  However, because we ultimately want to use xDS
to configure things via the service config, we also need to integrate
with xDS via the resolver.

## Implementation

We have been working on the implementation of this for a long time (and
it has gone through several design iterations in the process, evolving
as the implementation has progressed).  We expect it to be ready in
C-core, Java, and Go in the near future.
