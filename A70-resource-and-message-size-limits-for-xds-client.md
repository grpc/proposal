A70: Resource and message size limits for XdsClient
----
* Author(s): Easwar Swaminathan (@easwars)
* Approver: @markdroth
* Status: Draft
* Implemented in:
* Last updated: 2023-08-03
* Discussion at: TBD

## Abstract

xDS enabled gRPC clients and servers acquire their configuration by
communicating with an xDS management server. The XdsClient component inside gRPC
receives this configuration as xDS resources from the management server on a
single ADS stream. This document specifies gRPC's support for setting limits on
the size of these resources and the RPC messages which carry them.

## Background

xDS enabled gRPC clients and servers use an internal XdsClient component to
handle all communication with the xDS management server. The configuration
required to communicate with this management server is specified as part of the
xDS bootstrap configuration. This configuration includes the address of the
management server and the credentials to use when communicating with it among
other things. With support for federation, the bootstrap configuration could
specify more than one management server. The XdsClient creates one gRPC Channel
for every management server that it needs to communicate with.

The gRPC API enables users to configure the maximum allowed send and receive
message sizes on vanilla gRPC channels and servers. But there is no way
currently for users to specify the same for channels created by the XdsClient to
the management server. In fact,  the XdsClient does not configure these on the
channels created to the management server, and therefore they default to 4 MiB.
This document specifies a mechanism that will enable users to configure these
limits through new fields added to the bootstrap configuration.

It is important to note a few things about the XdsClient implementation here
(which affect memory utilization of the resource cache):
- it caches the raw serialized proto for all ACKed resources. This is required
  to support the xDS config dump via CSDS.
- it caches an internal representation of the ACKed resources containing *only*
  fields which are used by gRPC. This is required to ensure that received
  resources are deserialized only once, and are readily available to different
  components interested in these resources.
  - a deserialized proto usually consumes multiple times the memory of a
    serialized proto. Even though the cache does not store the complete
    deserialized proto (it stores only a small subset of fields used by gRPC),
    it needs to temporarily allocate enough memory to hold the complete
    deserialized proto at the time of deserialization.

gRPC currently supports only the State of the World (SotW) ADS variant of the
xDS protocol. This means that for LDS and CDS resources, the management server
must return all resources subscribed to by the XdsClient, in every response
message. We feel that the current value of 4MB for the maximum allowed message
size on the ADS stream is plenty. But there are users for whom this value is
either too small or too large. Making these configurable allows users to:
- receive larger messages from the management server, if and when it is
  appropriate
- protect themselves from a misconfigured or misbehaving management server

### Related Proposals:

* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A46: xDS NACK Semantics Improvement][A46]
* [gRFC A47: xDS Federation][A47]

[A27]: A27-xds-global-load-balancing.md
[A46]: A46-xds-nack-semantics-improvement.md
[A47]: A47-xds-federation.md

## Proposal

This design adds two new fields to the bootstrap config for controlling the:
- maximum allowed size for any single xDS resource received from the management
  server
  - xDS resources received from the management server are usually serialized
    inside of an `Any` proto and this limit applies to the size of the
    serialized `Any` proto
- maximum allowed size for any xDS message
  - An xDS message typically contains one or more resources, and this limit
    applies to the size of the serialized message

### Bootstrap Config Changes

Two new per-server fields that can be specified in the top-level server config
as well as per-authority server config, are defined as follows:

```
{
  "xds_servers": [
    {
      // Maximum allowed size in bytes for any single xDS resource received from
      // the management server. xDS resources received from the management server
      // are usually serialized inside of an Any proto and this limit applies to
      // the size of the serialized Any proto.
      "max_xds_resource_size": <size in bytes, if unspecified defaults to 4MiB>

      // Maximum allowed size for any xDS message received from the
      // management server. xDS messages typically contain more than one
      // resource and this limit applies to the size of the serialized message.
      "max_xds_message_size": <size in bytes, if unspecified defaults to 4MiB>
    }
  ],
}
```

### Temporary environment variable protection

As mentioned previously, the XdsClient currently does not impose any size limits
on the gRPC channel to the management server, and therefore gets the default
limit of 4MiB on the received message size. And if the above mentioned fields
are not set in the bootstrap configuration, the same behavior would continue and
therefore we don't need an environment variable to protect the changes described
here.

## Rationale

The approach described in this document serves all of our current needs and can
be implemented with a reasonable amount of time and effort. A whole bunch of
other options were considered, and are listed below:
- Use gRPC service config to specify maximum send and receive message sizes
  instead of introducing new fields in the bootstrap configuration.
  - Since the name resolver is under our control when the `xds:///` scheme is
    used, this is doable on xDS enabled gRPC clients. But, we also need this
    feature on xDS enabled gRPC servers and we won't have a name resolver or
    service config in that case.
- Do not make these values configurable, but instead set them to 2GB (which is
  the maximum size of a protobuf message).
  - Making these configurable does allow xDS enabled clients and servers to
    protect themselves from misconfigured and/or misbehaving control planes.
- Support a limit on the total resource cache size in the XdsClient.
  - It is difficult to define the expected behavior when the size limit is
    exceeded.
  - It is not ideal to impose limits on the number of resources referenced by
    another resource (e.g number of clusters in a route configuration, or the
    number of endpoints in a locality in LEDS), and such a client side limit is
    better handled with support from the xDS transport protocol and this will
    take time and will involve a bunch of other trade-offs.
- Support a config-validator extension point like what Envoy does. With this
  approach, we could write a validator implementation that performs an arbitrary
  computation to decide whether the resources received as part of an update are
  valid and NACK the entire update if one or more resources are considered
  invalid. But this is not the behavior we would want, for the same reason as
  described in [gRFC A46](A46-xds-nack-semantics-improvement.md).
- Support per-resource-type size limit instead of a global resource size limit.
  - There is no good known use case where a user would want to enforce different
    limits for different resource types.
- Support global message size limit instead of per-server.
  - The latter can be implemented with the same effort as the former, and gives
    us more flexibility.

## Implementation

The following needs to be implemented in the XdsClient
- Parse the new fields from the bootstrap configuration and plumb them down to
  the place where the gRPC channel to the management server is created.
- For enforcing the message size limit, all gRPC languages already support a
  call option to set the maximum receive message size. This will be set on the
  ADS stream to the management server. Some languages support default call
  options which can be specified at the time of channel creation instead of
  during every call, and this could be used if available.
  - If a received message is larger than the configured limit, the ADS stream
    must fail with a status code of `codes.ResourceExhausted`.
- For enforcing the per-resource size limit, we can rely on the size of the
  serialized bytes in the Any proto corresponding to the received resource.
  - If a resource exceeds the configured size limit, the response must be
    NACKed, while continuing to use other good resources in the same response.



