xDS Configuration Dump via Client Status Discovery Service in gRPC
----
* Author(s): lidizheng
* Approver: markdroth
* Status: Approved
* Implemented in: TBD
* Last updated: 2021-02-26
* Discussion at: https://groups.google.com/g/grpc-io/c/zL45YyxtJ08

## Abstract

[Client Status Discovery
Service](https://github.com/envoyproxy/envoy/blob/main/api/envoy/service/status/v3/csds.proto)
(CSDS) is a service that exposes xDS config of a given client. It’s commonly
used to query control planes for the synced xDS config of a particular sidecar
proxy. However, it can also be used to query an xDS-compliant application for
its received xDS configuration. This doc proposes a solution to implement a CSDS
service in gRPC, so our users can debug their service mesh easily.


## Background

Envoy [started](https://github.com/envoyproxy/envoy/pull/9383) the CSDS
development process in Dec 2019. The CSDS API has been available since xDS v2,
and it’s under active development. But, Envoy proxies do not support or
understand this service, only the control plane does. Envoy already provides
config dump via its [admin
interface](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) for
years. There is no visible plan for Envoy to support serving CSDS directly as a
proxy (see [envoy#13181](https://github.com/envoyproxy/envoy/issues/13181)).

The CSDS service accepts two methods, one for unary, one for streaming. The
expected behavior of the two methods is the same, but the streaming method can
reuse the stream. According to the service description below, this is not a
watch-style API.

```proto
// CSDS is Client Status Discovery Service. It can be used to get the status of
// an xDS-compliant client from the management server's point of view. It can
// also be used to get the current xDS states directly from the client.
service ClientStatusDiscoveryService {
  rpc StreamClientStatus(stream ClientStatusRequest) returns (stream ClientStatusResponse) {
  }

  rpc FetchClientStatus(ClientStatusRequest) returns (ClientStatusResponse) {
  }
}
```

### Related Proposals:
* [A27 - xDS-Based Global Load
  Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
* [A38 - Admin Interface API](https://github.com/grpc/proposal/pull/218)


## Proposal

### CSDS in gRPC

The goal of adding a CSDS service to gRPC is to enable programmatic access to
the operating xDS configs of a running gRPC application. To make this more
clear, an xDS-compliant application receives many xDS configs, which may be
rejected or ignored or obsoleted. **The gRPC CSDS service should always return
the currently accepted xDS configs.**


### ADS Parsing Logic Update: Continue After First Error

Current gRPC ADS parsing logic is when the response parser observes an error in
the ADS response, it fails the entire message and aborts parsing.

To improve the debuggability of gRPC, gRPC needs to populate **the reason** and
**affected resources** when rejecting an update response. This information
requires the ADS parser to continue parsing past the first error. Here are the
expected behavior under scenarios:

* If the entire message won’t parse: no need to record anything in CSDS, since
  we don’t know what type of xDS config or what resources are being updated;
* If one resource won’t parse: don’t abort the parsing, record the error in the
  `error_state.details` field. The error string will be attached to other
  resources that are successfully recognized;
* If one resource has validation error: record the error, and attach the error
  string to all resources in the ADS update including itself.

Here is an example of NACK information handling with new ADS changes:

```
# Imagine we have endpoint A, B, C
EDS -> {A, B, C}, version 1
gRPC -> ACK
CSDS -> [
  {Endpoint A, ACK, version 1},
  {Endpoint B, ACK, version 1},
  {Endpoint C, ACK, version 1}
]

# The newer endpoint B contains parsing error
EDS -> {A, B}, version 2
gRPC -> NACK, Failed to parse endpoint B
CSDS -> [
  {Endpoint A, NACK, version 1, rejected version 2, rejected reason: Failed to parse endpoint B},
  {Endpoint B, NACK, version 1, rejected version 2, rejected reason: Failed to parse endpoint B},
  {Endpoint C, ACK, version 1}
]

# Accepted update will clean error states
EDS -> {B, C} version 3
gRPC -> ACK
CSDS -> [
  {Endpoint A, NACK, version 1, rejected version 2, rejected reason: Failed to parse endpoint B},
  {Endpoint B, ACK, version 3},
  {Endpoint C, ACK, version 3}
]
```


### xDS Config Error State

When an ADS response is rejected, gRPC should provide debug information via
CSDS. This is done via `UpdateFailureState` within the `config_dump.proto`,
which includes the version, timestamp, and the reason of the rejected update
(see [proto
definition](https://github.com/envoyproxy/envoy/blob/a4024a578b3f2611fe26229f5d0de99eb0c56895/api/envoy/admin/v3/config_dump.proto#L72)).


### CSDS Service Design

Ideally, the CSDS service should be reusable by our users, instead of just an
internal tool for the gRPC library. So, the CSDS service should be implemented
in wrapper languages and Java/Go. There are three major parts in the service
design:

* Cache resources in the global XdsClient;
* Collect additional metadata about the cached resource (update timestamp,
  version, etc.);
* Assemble cached resources into the gigantic config dump response.

Note that xDS config should include the `config.core.v3.Node` information in the
`envoy.service.status.v3.ClientConfig.node` field. Unlike other xDS configs,
this information is static and may require extra logic to integrate into the
CSDS service.

For Java and Go, it’s straightforward that this feature can be implemented
bottom-up in the same language, and it’s possible to avoid an additional copy of
the message itself. But Core needs to transport the collected xDS configs across
the language boundary. The API needs to convert proto messages into C-compatible
types (e.g. `char *`, either bytes or JSON).


#### Detail: xDS v2/v3/v4

CSDS was meant to be a RPC service made for xDS management servers before this
proposal. But the service itself has the potential to serve client status on xDS
clients. However, during development, we found several constraints about the
existing service protocol, hence we merged several updates (see above) to
improve the CSDS service to meet our standard. The updates are made to xDS v3
and xDS v4 (alpha) only, since xDS v2 is in deprecated state.

For xDS CSDS v2 (the RPC service itself), this doc proposes to not support, so
we only provide v3 CSDS service and CSDS client requesting for v2 service will
get UNIMPLEMENTED error.

There are edge cases that v2 and v3 xDS resources are mixed together. This doc
proposes to keep them as-is in the CSDS responses. So, the CSDS service can
accurately reflect the xDS configs received from the control plane.

#### Detail: Expose xDS Config As Is

To date, gRPC supports the majority of xDS config but not all. Alternatively, we
could only dump the config that gRPC understands. However, doing so will create
a behavior difference between the gRPC and the control plane. It might be
confusing to users that some fields are missing. Also, during the configuration
dump, Envoy also dumps its entire configuration even if it doesn’t understand
some of the configuration fields (think of fields for Envoy extensions). So, for
correctness and simplicity, we should **cache and expose the xDS config the way
gRPC receives them**.

#### Detail: Cache Lifecycle

gRPC doesn't use xDS messages directly, but interprets them into language-native
class/struct. The lifecycle of the cached xDS messages should be identical to
the interpreted structure in each stack, so there is no memory management logic
change needed.

#### Detail: Generated Node Information

gRPC loads Node information from a [bootstrap
file](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md#xdsclient-and-bootstrap-file),
which includes the identification that will be used in the control plane. gRPC's
xDS client also generates `user_agent_name`, `user_agent_version`, and
`client_features` before sending the `Node` information to the control plane.
This information can be very helpful to locate the culprit release and
debugging. This doc recommends the implementation to provide the
`ClientStatusResponse.ClientConfig.node` field with the `Node` information that
the control plane will receive. Here is an example of a filled `Node`:

```json
{
  "node": {
    "id": "c591240a-b3b6-4761-aada-3fa43ec6a852",
    "cluster": "cluster",
    "metadata": {
      "...": "..."
    },
    "locality": {
      "zone": "zone"
    },
    "userAgentName": "gRPC Java",
    "userAgentVersion": "1.35.1-SNAPSHOT",
    "clientFeatures": ["envoy.lb.does_not_support_overprovisioning"]
  }
}
```

#### Detail: Node Matching

The node matching mechanism is not designed for gRPC’s use case, but for
querying the control plane. Technically, gRPC application only serves as one xDS
node, so there is no need to differentiate with xDS client's status to reply.
However, gRPC should reject the request with Node Matcher that failed to match
with the Node information of the gRPC application. This feature has lower
priority. To preserve forward compatibility, this doc proposes to return an
[INVALID_ARGUMENT](https://github.com/grpc/grpc/blob/master/doc/statuscodes.md)
if the CSDS implementation observe an non-empty Node Matcher.


### Config Dump Design: GenericXdsConfig vs. PerXdsConfig

CSDS offers two ways to structure the xDS config dumps:

1. [`PerXdsConfig`](https://github.com/envoyproxy/envoy/blob/3975bf5dadb43421907bbc52df57c0e8539c9a06/api/envoy/service/status/v3/csds.proto#L92):
   group configs by their xDS type (e.g., all Cluster configs);
2. [`GenericXdsConfig`](https://github.com/envoyproxy/envoy/blob/3975bf5dadb43421907bbc52df57c0e8539c9a06/api/envoy/service/status/v3/csds.proto#L133):
   a flat list of xDS configs.

gRPC implemented `PerXdsConfig` support in C++/Java/Go/Python since gRPC
v1.37.x. However, now that the `GenericXdsConfig` field is available, we found
it allows easier programmatic access, and it removes the confusing per-xds-type
version info. `PerXdsConfig` is now deprecated for xDS v3.

The size of xDS configs may be significant. Supporting two CSDS standards is
technically possible but at the cost of performance and verbose output. So, gRPC
libraries will upgrade the existing `PerXdsConfig` config dump implementation to
the latest CSDS standard `GenericXdsConfig`.


## Alternatives

### Solution: Config Dump By Attaching An HTTP Server

Although it is the most straightforward way of implementing an admin interface,
Java doesn’t have a good enough built-in web server and it is challenging to
introduce yet another dependency into Core. Exposing application states via HTTP
will be challenging from engineering perspective.

### Solution: Config Dump via Channelz

Though only a subset of xDS resources is applicable to a channel, xDS configs
(listeners, routes, clusters, endpoints) are global resources to a gRPC
application. One unsolved problem is that if we choose to use CSDS to expose xDS
configs, **the users won’t have the ability to query the set of effective xDS
configs for a channel**. The CSDS is not designed for finer granularity than
individual processes.

gRPC supports sharing XdsClient across multiple channels, the boundary of each
channel’s xDS config will be blurry. Injecting xDS info into a channel tracing
service won’t be the right direction in the long term.

### Solution: Config Dump via File

This approach saves the active xDS configuration onto a memory-based file system
location, like `sysctl`, or Envoy Runtime. However, due to the lack of two-way
communication and the complexity of the new machinery, this solution could cost
more engineering resources but yield less functionality.


## Implementation

### CSDS Proto Updates

* [envoy#13121]: the config synchronization status was defined from the xDS
  management server point of view, this PR adds a set of ENUM to present the
  config synchronization status from a client-side view.
* [envoy#14689]: the CSDS focused on dumping the in-effective xDS configs, but
  we can do better to improve gRPC’s debuggability. This PR adds fields to
  config dump protos to allow CSDS to return information about the rejected
  update.
* [envoy#14900]: this PR adds two additional status to client configs, the
  REQUESTED and the DOES_NOT_EXIST ([read
  more](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist)).

### CSDS Implementation

* Core: https://github.com/grpc/grpc/pull/25038
* Golang: To be linked
* Java: To be linked

[envoy#13121]: https://github.com/envoyproxy/envoy/pull/13121

[envoy#14689]: https://github.com/envoyproxy/envoy/pull/14689

[envoy#14900]: https://github.com/envoyproxy/envoy/pull/14900
