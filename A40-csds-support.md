xDS Configuration Dump via Client Status Discovery Service in gRPC
----
* Author(s): lidizheng
* Approver: markdroth
* Status: Draft
* Implemented in: All languages
* Last updated: 2021-02-16
* Discussion at: TBD

## Abstract

[Client Status Discovery
Service](https://github.com/envoyproxy/envoy/blob/main/api/envoy/service/status/v3/csds.proto)
(CSDS) is a service that exposes xDS config of a given client. It’s commonly
used to query control planes for the synced xDS config of a particular sidecar
proxy. However, it can also be used to query an xDS-compliant application for
its received xDS configuration. This doc proposes a solution to implement a CSDS
servicer in gRPC, so our users can debug their service mesh easily.


## Background

Envoy [started](https://github.com/envoyproxy/envoy/pull/9383) the CSDS
development process in Dec 2019. The CSDS API has been available since xDS v2,
and it’s under active development. But, Envoy proxies do not support or
understand this service, only the control plane does. Envoy already provides
config dump via its [admin
interface](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) for
years. There is no visible plan for Envoy to support serving CSDS directly as a
proxy.


### Related Proposals: 
* [A38 - Admin Interface API](https://github.com/grpc/proposal/pull/218)

## Proposal

### CSDS in gRPC
The goal of adding a CSDS servicer to gRPC is to enable programmatic access to
the operating xDS configs of a running gRPC application. To make this more
clear, an xDS-compliant application receives many xDS configs, which may be
rejected or ignored or obsoleted. **The gRPC CSDS servicer should always return
the currently accepted xDS configs.**

### Config Status in gRPC

One critical field in CSDS responses is the config status. The config status is
an additional signal to help users debug, which indicates **the synchronization
state of the given xDS resources with the control plane**. CSDS designed config
statuses from a control plane point of view, and it only works at the
granularity of xDS config type.

This gRFC proposes adding following status enum to
([config_dump.proto](https://github.com/envoyproxy/envoy/blob/main/api/envoy/admin/v3/config_dump.proto).
`ClientResourceStatus` better represents the viewpoint of xDS clients and it can
be as specific as per individual xDS resources.

```protobuf
// Resource status from the view of a xDS client, which tells the synchronization
// status between the xDS client and the xDS server.
enum ClientResourceStatus {
  // Resource status is not available/unknown.
  UNKNOWN = 0;

  // Client requested this resource but hasn't received any update from management
  // server. The client will not fail requests, but will queue them until update
  // arrives or the client times out waiting for the resource.
  REQUESTED = 1;

  // This resource has been requested by the client but has either not been
  // delivered by the server or was previously delivered by the server and then
  // subsequently removed from resources provided by the server. For more
  // information, please refer to the :ref:`"Knowing When a Requested Resource
  // Does Not Exist" <xds_protocol_resource_not_existed>` section.
  DOES_NOT_EXIST = 2;

  // Client received this resource and replied with ACK.
  ACKED = 3;

  // Client received this resource and replied with NACK.
  NACKED = 4;
}
```

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
  error string. The error string will be attached to other resources that are
  successfully recognized;
* If one resource has validation error: record the error, and attach the error
  string to all resources including itself.

Here is an example of NACK information handling with new ADS changes:

```
# Imagine we have endpoint A, B, C
EDS -> {A, B, C}, version 1
gRPC -> ACK
CSDS -> {Endpoint A, version 1}, {Endpoint B, version 1}, {Endpoint C, version 1}

# The newer endpoint B contains parsing error
EDS -> {A, B}, version 2
gRPC -> NACK, Failed to parse endpoint B 
CSDS -> {Endpoint A, version 1, rejected version 2, rejected reason: Failed to parse endpoint B } {Endpoint B, version 1, rejected version 2, rejected reason: Failed to parse endpoint B} {Endpoint C, version 1}

# Accepted update will clean error states
EDS -> {B, C} version 3
gRPC -> ACK
CSDS -> {Endpoint A, version 1, rejected version 2, rejected reason: Failed to parse endpoint B } {Endpoint B, version 3}, {Endpoint C, version 3}
```


### xDS Config Error State

When an ADS response is rejected, gRPC should provide debug information via
CSDS. This is done via `UpdateFailureState` within the `config_dump.proto`,
which includes the version, timestamp, and the reason of the rejected update:

```proto
message UpdateFailureState {
  ...

  // Time of the latest failed update attempt.
  google.protobuf.Timestamp last_update_attempt = 2;

  // Details about the last failed update attempt.
  string details = 3;

  // This is the version of the rejected resource.
  string version_info = 4;
}
```

### CSDS Servicer Design

Ideally, the CSDS servicer should be reusable by our users, instead of just an
internal tool for the gRPC library. So, the CSDS servicer should be implemented
in wrapper languages and Java/Go. There are three major parts in the servicer
design:

* Cache resources in the global XdsClient;
* Collect additional metadata about the cached resource (update timestamp,
  version, etc.);
* Assemble cached resources into the gigantic config dump response.

For Java and Go, it’s straightforward that this feature can be implemented
bottom-up in the same language, and it’s possible to avoid an additional copy of
the message itself. But Core needs to transport the collected xDS configs across
the language boundary. The API needs to convert proto messages into C-compatible
types (e.g. `char *`, either bytes or JSON).

#### Detail: No xDS v2 Support

CSDS was meant to be a RPC service made for xDS management servers before this
proposal. But the service itself has the potential to serve client status on xDS
clients. However, during development, we found several constraints about the
existing service protocol, hence we merged several updates (see above) to
improve the CSDS service to meet our standard. The updates are made to xDS v3
and xDS v4 (alpha) only, since xDS v2 is in deprecated state. This doc proposes
to not support xDS v2 for CSDS.

#### Detail: Cache Lifecycle

gRPC doesn't use xDS messages directly, but interpret them into language-native
class/struct. The lifecycle of the cached xDS messages should be identical to
the interpreted structure in each stack, so there is no memory management logic
change needed.

#### Detail: Expose xDS Config As Is

To date, gRPC supports the majority of xDS config but not all. Alternatively, we
could only dump the config that gRPC understands. However, doing so will create
a behavior difference between the gRPC and the control plane. It might be
confusing to users that some fields are missing. Also, during the configuration
dump, Envoy also dumps its entire configuration even if it doesn’t understand
some of the configuration fields (think of fields for Envoy extensions). So, for
correctness and simplicity, we should **cache and expose the xDS config the way
gRPC receives them**.

#### Detail: Node Matching

The node matching mechanism is not designed for gRPC’s use case, but for
querying the control plane. This doc proposes to ignore the Node matching query.


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

If we decided to support sharing XdsClient across multiple channels, the
boundary of each channel’s xDS config will be blurry. Injecting xDS info into a
channel tracing service won’t be the right direction in the long term.

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
  REQUESTED and the DOES_NOT_EXIST (read more).

### CSDS Implementation

* Core: https://github.com/grpc/grpc/pull/25038
* Golang: To be linked
* Java: To be linked

[envoy#13121]: https://github.com/envoyproxy/envoy/pull/13121

[envoy#14689]: https://github.com/envoyproxy/envoy/pull/14689

[envoy#14900]:https://github.com/envoyproxy/envoy/pull/14900