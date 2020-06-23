A30: xDS v3 Support
----
* Author(s): Mark D. Roth (markdroth)
* Approver: ejona86, dfawley
* Status: Draft
* Implemented in: 
* Last updated: 2020-06-23
* Discussion at: 

## Abstract

The purpose of this proposal is to describe how gRPC will handle the
transition from v2 to v3 of the xDS API and to document how gRPC will handle
xDS versioning on an ongoing basis.

## Background

gRPC has recently added support for obtaining configuration via the
[xDS API](https://github.com/envoyproxy/data-plane-api).  Our current
implementation uses uses v2 of the xDS API, but we are now adding
support for [xDS
v3](https://docs.google.com/document/d/1xeVvJ6KjFBkNjVspPbY_PwEDHC7XPi0J5p1SqUXcCl8/edit).
In addition, we have been working with the xDS community on the new
[minor/patch versioning
proposal](https://docs.google.com/document/d/1afQ9wthJxofwOMCAle8qF-ckEfNWUuWtxbmdhxji0sI/edit),
which should provide a smoother transition mechanism going forward.

Regardless of whether a new version is indicated by bumping the
major version (as was done with the v2 to v3 transition) or by bumping
the minor version (as will probably be done going forward), the current
plan is that there will still be one new version of the xDS API per
year.

### Related Proposals: 

[A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)

## Proposal

### Version Support Policy

gRPC's policy will be to support two versions of xDS at any given time.
So if gRPC currently supports versions N-1 and N, then when xDS version
N+1 is released, gRPC will add support for version N+1 and drop support
for version N-1 at the same time, so that we will support versions N
and N+1.

This policy will give users 1-2 years to upgrade their xDS servers in
order to continue using the latest release of gRPC.  Older releases of
gRPC will of course continue to function using the older versions of the
xDS API.

### Deciding What Version to Use

Any given gRPC release will support two versions of xDS, versions N-1 and N.
It will assume that all servers speak version N-1, and there will be a
server feature defined in the bootstrap file (see below) to tell it that
the server supports version N.  When speaking to the server, gRPC will use
the highest version that is common to both itself and the server.

For example, let's say that the gRPC binary supports versions N-1 and N.
If there is no server feature in the bootstrap file telling it that the
server supports version N, it will assume that the server supports only
version N-1, and it will speak version N-1 to the server.

However, if the bootstrap file does contain a server feature telling gRPC
that the server supports version N, then it will believe that the server
supports both versions N-1 and N, and it will speak version N to the server.

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

For each new version of xDS, a server feature will be defined.  The
presence of that feature in the `server_features` list will indicate
that the server supports that version of xDS.

Note that because each release of gRPC will support only two versions of
xDS and will always assume that servers support the lower of the two
versions, a server feature added to support version N+1 will no longer
be used once gRPC adds support for version N+2.  However, it will not
cause any problems to be left in the bootstrap file.  So at any given
time, it useful to have a bootstrap file indicating the total set of
versions supported by your xDS server, and each client will use the
latest version it supports.

For xDS v3, the server feature will be the string `xds_v3`.

### Transition Examples

Let's say that your xDS server currently supports versions N-1 and N.
Your bootstrap files can contain the server features for both of those
versions (if applicable -- there is no server feature for xDS v2).  This
allows the following:

- Older gRPC clients that support versions N-2 and N-1 will see the
  server feature for version N-1 in the bootstrap file, so they will use
  version N-1.
- Newer gRPC clients that support versions N-1 and N will see the server
  feature for version N in the bootstrap file, so they will use version
  N.  (If for some reason the server feature for version N was not
  present in the bootstrap file, they would also work using version N-1.
  Note that these clients will not care whether the bootstrap file
  contains the server feature for version N-1, because they will assume
  that the server supports this version anyway.)

Now let's say that you want to drop support for version N-1 in your xDS
server, so that it supports only version N.  Before you can do that,
you need to upgrade the older gRPC clients that speak only versions N-2
and N-1.  You have two choices:

- Upgrade the clients to a release supporting versions N-1 and N.  These
  clients will use version N by virtue of the server feature for version
  N in the bootstrap file.
- Upgrade the clients to the newest release supporting versions N and
  N+1.  These clients will not see a server feature for version N+1 in
  the bootstrap file, so they will use version N.  (They will not
  actually care whether the server feature for version N is present in
  the bootstrap file, since they will assume the server supports this
  version anyway.)

Once you are sure that you have no more clients using version N-1, you
can drop support for that version from your xDS server.

Next, let's say that xDS version N+1 has been released, and you want to add
support for that version to your xDS server.  You can of course do this at
any time, but none of the existing gRPC clients will actually use it
unless you take additional steps.

- For gRPC clients that support versions N and N+1, all you need to do
  is add the server feature for version N+1 to the bootstrap file.  That
  will cause these clients to start using version N+1.
- For older gRPC clients that support versions N-1 and N, they will
  ignore the server feature for version N+1 in the bootstrap file, so
  they will continue to use version N.  You will need to upgrade to a
  more recent release of gRPC to get them to use version N+1.

Note that the only case in which users will need to upgrade their gRPC
binaries in lock step with their xDS server is if they are migrating
their xDS server from version N-1 to version N+1 in a single step,
without adding support for version N in between.  We do not recommend
this kind of transition.

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

Luckily, we can be more flexible with the resource version, because
there's essentially only one case where we use a field in the v2 protos
that was removed in the v3 protos.  (That case is the node `build_version`
field, which we can handle as a special case.)  Therefore, we can
simply change our code to parse all incoming messages as v3, regardless
of whether they are actually v2 or v3 messages.  We will also need to
accept both v2 and v3 type names interchangeably in all `type_url` fields.

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

### Version of LRS Protocol

TODO(roth): Need to figure this out.  We had been planning to use the
`transport_api_version` field in the LRS `ConfigSource` in the CDS
response to set this.  However, that `ConfigSource` contains a
`SelfConfigSource`, not an `ApiConfigSource`, so there is no
`transport_api_version` field.  One option is to add a `transport_api_version`
field to `SelfConfigSource`.  Another option would just be to say that
the server feature in the bootstrap file controls both the xDS and LRS
transport versions, although I'm not sure whether that will be flexible
enough in the long run.

### Environment Variable

Initially, support for xDS v3 in gRPC will be considered experimental,
since we are planning to release this support in the client before we
have been able to perform interop testing with any real-world v3 xDS
server.  As a result, we are taking the added safety precaution that the
new server feature in the bootstrap file will only be used if the
`GRPC_XDS_EXPERIMENTAL_V3_SUPPORT` environment variable is set to
`true`.  This environment variable guard will be removed once we have
performed the necessary interop testing.

For future version transitions, we hope to be able to perform interop
testing before we release support for the new version, in which case
this kind of environment variable protection will not be needed.

## Rationale


## Implementation

C-core implementation in progress in https://github.com/grpc/grpc/pull/23281.
