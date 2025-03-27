A53: Option for Ignoring xDS Resource Deletion
----
* Author(s): Mark D. Roth (@markdroth)
* Approver: @ejona86, @dfawley
* Status: In Review
* Implemented in: C-core, Java
* Last updated: 2022-05-19
* Discussion at: https://groups.google.com/g/grpc-io/c/eh7En9CY3oY
* Deprecated by: [A88: xDS Data Error Handling](A88-xds-data-error-handling.md)

## Abstract

This design specifies an option for the gRPC XdsClient that allows it to
ignore resource deletions sent by the xDS server, thus making it more
fault tolerant in the face of control plane bugs or misconfigurations
that delete required resources out from under gRPC.

## Background

Currently, when an xDS server tells a client that an LDS or CDS resource
has been deleted, gRPC will unconfigure the behavior that had been
configured by that resource.  For example, on the gRPC client, an LDS
resource deletion will put the channel in `TRANSIENT_FAILURE`, and a CDS
resource deletion will cause the channel to fail RPCs bound for that
cluster.  On the gRPC server, an LDS resource deletion will cause the
server to stop accepting incoming connections, effectively stopping
the server.

This behavior means that gRPC will not behave in a fault-tolerant manner
in the face a control plane bug or misconfiguration that deletes a
resource that it was actually using, which can lead to an outage.  To avoid
this, we would like gRPC to ignore the deletion of any required resource.

However, if the control plane does not have a mechanism to alert human
operators when a client is requesting a resource that does not exist,
then it is not necessarily safe for the client to simply ignore these
deletions.  This could lead to a situation where the control plane
deploys a bad update, but the operators think everything is fine because
the clients are still working, and then a problem is discovered days or
even months later when the clients are restarted and fail to come back up.
This could lead to a different type of outage than the one that this design
is intended to address.

Because we cannot know whether a given control plane has appropriate
alerting for human operators when a client is requesting a resource that
does not exist, we propose to make this behavior configurable in gRPC.

### Related Proposals: 

* [gRFC A27: xDS-Based Global Load Balancing](A27-xds-global-load-balancing.md)
* [A30: xDS v3 Support](A30-xds-v3.md)
* [A47: xDS Federation](A47-xds-federation.md)

## Proposal

The ability to ignore resource deletions from the xDS server will be
configured using the server feature mechanism introduced in [gRFC
A30](A30-xds-v3.md).  Note that, as per [gRFC A47](A47-xds-federation.md),
a separate server feature list can be configured for each server in a
federated configuration, so this feature can be enabled on a per-server
basis.

The new server feature will be the string `ignore_resource_deletion`; if
present, this feature will be enabled.

When this feature is enabled, the XdsClient will not invoke the
watchers' `OnResourceDoesNotExist()` method when a resource is deleted,
nor will it remove the existing resource value from its cache.

Currently, this behavior will affect only LDS and CDS, because those are
the only two resource types for which the state-of-the-world xDS protocol
variant allows the server to indicate a deletion to a client.  However,
in the future, if/when gRPC adds support for the incremental xDS
protocol variant, this behavior would affect all resource types.

The XdsClient will generate a WARNING log (may be ERROR on
implementations that don't support WARNING) when a resource deletion is
ignored and a subsequent INFO log message when the condition clears, either
because the server once again started sending the resource or because
the client has unsubscribed from the resource.  To implement this, the
XdsClient will maintain a bit in the cache entry for each resource
indicating whether a deletion has been ignored.  When the XdsClient
receives an LDS or CDS update that does not include the resource, if the
bit is not set, it will log the ERROR message and set the bit; when the
XdsClient receives an LDS or CDS update that includes a resource for
which the cache entry bit is set, it will log the INFO message and unset the
bit; when the XdsClient unsubscribes from a resource, if the cache entry bit
is set, it will log the INFO message before deleting the cache entry.

### Temporary environment variable protection

This behavior is not configured via remote interaction, older clients
will ignore unknown fields in the bootstrap config, and the feature will
be implemented in a single atomic change (i.e., there will not be any
client in which the functionality is only partially present), and is
not expected to need interop testing, so it does not require environment
variable protection.

## Rationale

As mentioned above, we cannot make this behavior unconditionally
enabled, because it may lead to an outage due to human operators being
blind to a bad push until clients are restarted.

For the logging behavior, we considered logging on every LDS or CDS
response in which the resource is not present, even if the resource
was already absent in the previous response.  However, we decided
against this, because when we eventually add support for the incremental
xDS protocol variant, it will no longer be possible to log repeatedly,
so we don't want users to get used to a behavior that we will not be
able to continue to provide in the future.

## Implementation

 * C-core implementation in https://github.com/grpc/grpc/pull/29633 released in
   v1.48.0
 * Java implementation in https://github.com/grpc/grpc-java/pull/9339 released
   in v1.48.0

## Open issues (if applicable)

N/A
