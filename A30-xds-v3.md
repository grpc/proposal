A30: xDS v3 Support
----
* Author(s): Mark D. Roth (markdroth)
* Approver: ejona86, dfawley
* Status: Final
* Implemented in: C-core, Java, Go, Node
* Last updated: 2022-10-11
* Discussion at: https://groups.google.com/g/grpc-io/c/YRVqRMUQwhM and
  https://groups.google.com/g/grpc-io/c/pfwDmD7cpp0

## Abstract

The purpose of this proposal is to describe how gRPC will handle the
transition from v2 to v3 of the xDS API.

## Background

gRPC has recently added support for obtaining configuration via the
[xDS API](https://github.com/envoyproxy/data-plane-api).  Our current
implementation uses v2 of the xDS API, but we are now adding support
for [xDS
v3](https://docs.google.com/document/d/1xeVvJ6KjFBkNjVspPbY_PwEDHC7XPi0J5p1SqUXcCl8/edit).

### Related Proposals: 

[A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)

## Proposal

### Migration Timing

We will give users 1-2 years to upgrade their xDS servers to xDS v3 in
order to continue using the latest release of gRPC.  Older releases of
gRPC will of course continue to support xDS v2.

### Deciding What Version to Use

There will be a server feature defined in the bootstrap file (see below)
to indicate that the server supports xDS v3.  If this server feature is
set in the bootstrap file for a given server, then when speaking to the
server, gRPC will use xDS v3; if the server feature is not present, then
gRPC will use xDS v2.

In the future, when we drop support for xDS v2, gRPC will stop looking
at the server feature in the config file, and it will unconditionally
use xDS v3.

### Server Features in the Bootstrap File

The bootstrap file that configures the XdsClient code in gRPC will have
a new `server_features` field added to it.  The new format will be as
follows:

```
{
  // The xDS server to talk to.  The value is an array to allow for a
  // future change to add support for failing over to a secondary xDS server
  // if the primary is down, but for now, only the first entry in the
  // array will be used.
  "xds_servers": [
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
      ],

      // THIS FIELD IS NEW!
      // A list of features supported by the server.  New values will
      // be added over time.  For forward compatibility reasons, the
      // client will ignore any entry in the list that it does not
      // understand, regardless of type.
      "server_features": [ ... ]

    }
  ],
  "node": <JSON form of Node proto>
}
```

For xDS v3 support, the server feature will be the string `xds_v3`.  The
presence of that feature in the `server_features` list will indicate
that the server supports xDS v3.

### Details of the v2 to v3 Transition

The xDS major version number appears in two main places in the API:

- The RPC method name used to make the xDS request.  For v2, this is
  `/envoy.service.discovery.v2.AggregatedDiscoveryService/StreamAggregatedResources`,
  whereas for v3 it is
  `/envoy.service.discovery.v3.AggregatedDiscoveryService/StreamAggregatedResources`.
  We call this the *transport protocol version*.
- The type of resources that we request via the transport protocol.  For
  example, the type for a Listener resource in v2 is `envoy.api.v2.Listener`,
  whereas in v3 the type is `envoy.config.listener.v3.Listener`.  We
  call this the *resource version*.

When gRPC uses v3 to talk to a server, the primary thing that it is
choosing is the transport protocol version, not the resource version.

Luckily, we can be more flexible with the resource version, because there
are no cases where we use a field in the v2 protos that was removed in
the v3 protos.  Therefore, we can simply change our code to parse all
incoming messages as v3, regardless of whether they are actually v2
or v3 messages.  We will also need to accept both v2 and v3 type names
interchangeably in all `type_url` fields.

Note that we will actually take the same approach for the proto messages
related to the transport version, such as `DiscoveryRequest` and
`DiscoveryResponse`.  We will unconditionally use the v3 protos to
serialize and deserialize these messages, regardless of what version of
the transport protocol we are using.  There is one case where we are
using a field in the v2 protos that does not exist in the v3 protos,
which is the `build_version` field; for this one special case, we will
use a hack to manually populate this field in the v3 proto message by its
v2 field number when we are speaking the v2 transport version.

This means that the logic for determining the transport protocol version
is completely independent of the logic for handling the resource version.
gRPC will always request resources of the same version as the transport
protocol it is using.  However, it will accept any supported version of
resources in responses from the server, regardless of what version of
the transport protocol we are using.

In other words, if we're using the v2 transport protocol, we will
request v2 resources, but it's fine for the server to send us either v2
or v3 resources.  Similarly, if we're using the v3 transport protocol,
we will request v3 resources, but it's fine for the server to send us
either v2 or v3 resources.

Note that the ability to accept resource versions that are different
than what was requested is only intended for use as a rollout strategy
during transitions where the management server has some a priori
knowledge that its clients can support resources of a different version.
In the general case, management servers cannot assume this, because
(e.g.) if a client requests v2 resources, it might not be able to handle
v3 resources sent in response.

### Version of LRS Protocol

For now, we will use the same server feature in the bootstrap file to
indicate both the xDS and the LRS transport protocol version to use.

Note that this means that there will not be a way to configure the client
to use different transport versions for the two protocols.  In the future,
if this becomes a problem, we can make use of `transport_api_version`
field being added in https://github.com/envoyproxy/envoy/pull/11754 and
https://github.com/envoyproxy/envoy/pull/11824 to specify the transport
version of LRS.

Just as with xDS, the LRS client will parse responses as v3, regardless
of which version of the transport protocol is being used.

### Environment Variable

Initially, support for xDS v3 in gRPC will be considered experimental,
since we are planning to release this support in the client before we
have been able to perform interop testing with any real-world v3 xDS
server.  As a result, we are taking the added safety precaution that the
new server feature in the bootstrap file will only be used if the
`GRPC_XDS_EXPERIMENTAL_V3_SUPPORT` environment variable is set to
`true`.  This environment variable guard will be removed once we have
performed the necessary interop testing.

## Rationale


## Implementation

C-core implementation in https://github.com/grpc/grpc/pull/23281.
