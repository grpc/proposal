A88: xDS Data Error Handling
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-03-11
* Discussion at: https://groups.google.com/g/grpc-io/c/Gg_tItYgQoI

## Abstract

This document proposes adding support for the new xDS error propagation
mechanism described in [xRFC TP3], which provides a cleaner mechanism
for detecting non-existing resources.  It also defines improved behavior
for handling all xDS-related data errors, including improved behavior
for the xDS resource timer.

## Background

### Reporting Errors from the xDS Server

Currently, the xDS transport protocol does not provide a way for the
server to report per-resource errors to the client.  The only thing that
the server can do is react as if the resource does not exist, which
means that the client lacks an informative error message that could be
used for debugging purposes.

[xRFC TP3] provides a mechanism for the xDS server to send per-resource
errors to the client.  This will cover both cases like the resource not
existing (which also allows the client to detect this case more quickly,
as described below) and other errors, such as permission problems.

### xDS Error Handling

There are two basic categories of errors that can occur in an xDS client:
- **Transient errors:** These are connectivity problems reaching the
  xDS server, such as the xDS channel reporting TRANSIENT_FAILURE or the
  ADS stream failing without having received a response from the server,
  as described in [gRFC A57].  It also includes errors reported by the
  control plane via the new mechanism described in [xRFC TP3] with a
  status code other than NOT_FOUND or PERMISSION_DENIED.
- **Data errors:** This includes things like resource validation failures
  that cause the client to NACK an update and resource deletions sent
  by the xDS server.  It also includes errors reported by the control
  plane via the new mechanism described in [xRFC TP3] with a status code
  of NOT_FOUND or PERMISSION_DENIED.

For either category of error, if the error occurs when we do not yet
have a cached version of the resource, then we have no choice but to
fail data plane RPCs.  That is already our behavior today, and this
proposal does not modify that behavior.

For transient errors, if we do already have a previously cached version
of the resource, the desired behavior is to ignore the error and keep
using the previously cached version of the resource.  This is already
our behavior today, and this proposal does not modify that behavior.

However, for data errors, if we do already have a previously cached
version of the resource, our current behavior is somewhat inconsistent
and ad hoc.  It is useful to consider the desired behavior here from
first principles.  There are two main approaches that we can take here:

1. We can ignore the error and keep using the previously cached resource,
   in effect treating it as a transient error.  This approach is safer
   from the perspective of helping to avoid an immediate outage.  However,
   it does rely on the control plane identifying the problem and alerting
   humans to take action, or else the problem becomes completely silent,
   which might simply defer the outage until some arbitrary amount of
   time later when the clients are restarted, at which point it will be
   harder to debug.

2. We can throw away the previously cached resource and act as if we
   never saw it, thus failing data plane requests.  This makes the problem
   immediately obvious to human operators but may cause a user-visible
   outage.

Because different xDS servers may have different capabilities in terms
of how effectively they can alert humans to take action on data errors,
we cannot know which of those two approaches we should take for a given
xDS server.  Therefore, a choice between these two approaches is ideally
something that should be configurable on a per-server basis.

Unfortunately, our current behavior for data errors does not follow the
principles outlined above.  Currently, we have special handling for
the resource deletion case: by default, we throw away the previously
cached resource (i.e., approach 2 above), but we provide an option
in the bootstrap config to inhibit this behavior (see [gRFC A53]),
which confusingly makes the deletion not show up at all in CSDS (see
[gRFC A40]) or XdsClient metrics (see [gRFC A78]).  However, all other
data errors are actually treated as transient errors: we unconditionally
stick with the previously cached resource (i.e., approach 1 above).

This proposal changes the data error handling behavior to match the
principles outlined above.

### The Resource Timer

The xDS State of the World (SotW) protocol does not currently
provide a way for servers to explicitly inform clients when a
subscribed resource does not exist, so clients have to impose a
15-second does-not-exist timer for each individual resource when
the client initially subscribes to it, as described in the [xDS protocol
documentation](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist).
The exact semantics of this resource timer are tricky to get right,
as described in [gRFC A57].[^1]

[^1]: Note that gRFC A57 primarily addresses transient errors, while the
      does-not-exist timer actually reflects a data error.  However,
      the two needed to be addressed together, since we want the timer
      to run only when there is no transient error condition.

We have seen a number of cases where the resource timer has triggered
in production.  These cases have all been very hard to debug due to
the lack of a better error message, and in almost all cases there was
at least the suspicion that the timer triggered even when the resource
really did exist due to the xDS server simply being slow.

As a result of these problems, we want to work toward eliminating this
resource timer in the client in the long term.  As a first step toward
that goal, we propose some concrete changes to minimize the impact of
this timer.

A key part of this is adding support for [xRFC TP3], which proposes
a mechanism for the xDS server to communicate errors to clients in
cases where the server is unable to send a resource that the client has
subscribed to.

### Related Proposals: 
- [gRFC A53: Option for Ignoring xDS Resource Deletion][gRFC A53]
- [gRFC A57: XdsClient Failure Mode Behavior][gRFC A57]
- [gRFC A40: xDS Configuration Dump via Client Status Discovery Service][gRFC A40]
- [gRFC A78: OTel Metrics for WRR, Pick First, and XdsClient][gRFC A78]
- [gRFC A54: Restrict Control-Plane Status Codes][gRFC A54]
- [gRFC A61: IPv4 and IPv6 Dualstack Backend Support][gRFC A61]
- [gRFC A74: xDS Config Tears][gRFC A74]
- [xRFC TP3: Error Propagation][xRFC TP3]

[gRFC A53]: A53-xds-ignore-resource-deletion.md
[gRFC A57]: A57-xds-client-failure-mode-behavior.md
[gRFC A40]: A40-csds-support.md
[gRFC A78]: A78-grpc-metrics-wrr-pf-xds.md
[gRFC A54]: A54-restrict-control-plane-status-codes.md
[gRFC A61]: A61-IPv4-IPv6-dualstack-backends.md
[gRFC A74]: A74-xds-config-tears.md
[xRFC TP3]: https://github.com/cncf/xds/blob/main/proposals/TP3-xds-error-propagation.md

## Proposal

This proposal has the following parts:
- changes in the XdsClient watcher APIs
- handling data errors and transient errors
- support for errors provided by the xDS server as per [xRFC TP3]
- changes to the resource timer behavior
- removing the "ignore_resource_deletion" server feature
- new cache entry states, reflected in CSDS and XdsClient metrics

### Changes to XdsClient Watcher APIs

The current `XdsClient` API has three methods:

- `OnResourceChanged(ResourceType resource)`: Invoked whenever a new
  version of the resource is received from the xDS server.

- `OnError(Status status)`: Invoked for all transient errors and for some
  data errors (NACKs).  The watcher is generally expected to ignore the
  error if it already has a valid cached resource.

- `OnResourceDoesNotExist()`: Invoked specifically for the does-not-exist
  case.  The watcher is generally expected to stop using any previously
  cached resource and put itself into a failing state.
  - Note: The mechanism described in [gRFC A53] inhibits these
    notifications for only the case where we already had a cached version
    of the resource, which in the SotW protocol applies only to LDS
    and CDS.  That mechanism does not affect the case where we do not
    already have a cached version of the resource, which is where the
    does-not-exist timer is used.

These methods do not map well to the desired error handling behavior
defined above.

This proposal replaces those methods with the following two methods:

- `OnResourceChanged(StatusOr<ResourceType> resource)`: Will be invoked
  to notify the watcher of a new version of the resource received from
  the xDS server or an error indicating the reason why the resource cannot
  be obtained.  Note that if an error is passed to this method after the
  watcher has previously seen a valid resource, the watcher is expected
  to stop using that previously delivered resource.  In this case, the
  XdsClient will remove the resource from its cache, so that CSDS (see
  [gRFC A40]) and XdsClient metrics (see [gRFC A78]) will not reflect a
  resource that the client is not actually using.

- `OnAmbientError(Status status)`: Will be invoked to notify the watcher
  of an error that occurs after a resource has been received that should
  not modify the watcher's use of that resource but that may be useful
  information about the ambient state of the XdsClient.  In particular,
  the watcher should *not* stop using the previously seen resource, and the
  XdsClient will *not* remove the resource from its cache.  However, the
  error message may be useful as additional context to include in errors
  that are being generated for other reasons.  For example, if failing a
  data plane RPC due to all endpoints being unreachable, it may be useful
  to also report that the client has lost contact with the xDS server.
  The error condition is considered to be cleared when either
  `OnResourceChanged()` is invoked or `OnAmbientError()` is invoked
  again with an OK status.

### Handling Data Errors

As described above, the behavior that we want on data errors depends
on whether we can depend on the control plane to properly alert human
operators when these errors occur.  To configure this, we will introduce a
new server feature in the bootstrap config called "fail_on_data_errors",
which indicates that we cannot depend on the control plane to properly
alert human operators.

This server feature affects all data errors that can occur when a
resource is already cached, which are the following cases:
- NACKs
- resource deletions in LDS and CDS
- [xRFC TP3] errors with status NOT_FOUND or PERMISSION_DENIED

When one of these errors occurs, if the "fail_on_data_errors" server
feature is present in the bootstrap config, we will delete the existing
cached resource, if any.  Then, if there is an existing cached resource
(which there will not be if we just deleted it), call the watchers'
`OnAmbientError()` method with the error.  Otherwise, call the watchers'
`OnResourceChanged()` method with the error.

Note that the status code passed to the watcher should not be
directly propagated to data plane RPCs.  This is particularly true
for NOT_FOUND, which gRPC is prohibited from generating on a data
plane RPC.  For details, see [Propagating Status to Data Plane
RPCs](#propagating-status-to-data-plane-rpcs) below.

### Handling Transient Errors

Transient errors include the following cases:
- the xDS channel reporting TRANSIENT_FAILURE or the ADS stream failing
  without having received a response from the server, as described in
  [gRFC A57]
- [xRFC TP3] errors with any status *except* NOT_FOUND or PERMISSION_DENIED

When one of these errors occurs, if there is an existing cached resource,
we will call the watchers' `OnAmbientError()` method with the error.
Otherwise, we will call the watchers' `OnResourceChanged()` method with
the error.

Note that the status code passed to the watcher should not be directly
propagated to data plane RPCs.  For details, see [Propagating Status to
Data Plane RPCs](#propagating-status-to-data-plane-rpcs) below.

### New Watchers Started After Ambient Errors

Note that if a new watcher is started for a resource that already has a
watcher, and the original watcher has seen both a valid resource and an
ambient error, then the newly started watcher should also see both the
valid resource and the ambient error, in that order.

### Support for Errors Provided by the xDS server as per [xRFC TP3]

We will add support for the new response fields added in [xRFC TP3].
There will be no configuration telling gRPC to use these fields; we will
unconditionally use them if they are present.

When we receive an error for a given resource, we will do the following:
1. Cancel the resource timer, if any.
2. Set the cache entry's status to `RECEIVED_ERROR` and record the error
   in the cache entry.
3. If the status code is NOT_FOUND or PERMISSION_DENIED and the
   "fail_on_data_errors" server feature is present in the bootstrap
   config, delete the existing cached resource, if any.
4. If there is an existing cached resource (which there will not be if
   it was deleted in the previous step), call the watchers'
   `OnAmbientError()` method with the error.  Otherwise, call the
   watchers' `OnResourceChanged()` method with the error.

### Changes to Resource Timer Behavior

If we know that the xDS server will use [xRFC TP3] to inform us of
non-existing resources, then we no longer need to rely on the resource
timer to detect non-existing resources, so we can instead re-purpose
the timer to detect slow xDS servers.

To that end, we will introduce a new server feature in the bootstrap
config called "resource_timer_is_transient_error".  When this server
feature is present, the timer will be set for 30 seconds instead of 15
seconds, and when the timer fires, the resulting error will have status
UNAVAILABLE instead of NOT_FOUND, and the cache state will be set to
`TIMEOUT` instead of `DOES_NOT_EXIST`.

Note that regardless of whether this server feature is present, we will
still invoke the watchers' `OnResourceChanged()` method with the error,
because by definition this timer will fire only when we do not have a
version of the resource cached.

This server feature should be used only in cases where the server is known
to support [xRFC TP3].  If the server feature is used when the server
does *not* support [xRFC TP3], then it will take longer for the client
to notice a non-existing resource, because the server has no way of
reporting the condition before the timer fires.

Note that the client will honor [xRFC TP3] errors returned by the server
regardless of whether this server feature is present.  The presence of
this server feature actually controls only whether we can *depend* on the
server to provide these errors, so that we can be more lax about the
resource timer.

### Deprecating "ignore_resource_deletion" Server Feature

Because we now want the ability to control the behavior of all data
errors consistently rather than special-casing resource deletion, we
will deprecate the "ignore_resource_deletion" server feature that was
added in [gRFC A53].

For existing bootstrap configs that have the "ignore_resource_deletion"
server feature, gRPC will ignore that server feature, but the new default
behavior is to treat all data errors as transient, including resource
deletion, so they will still get the right behavior.

For existing bootstrap configs that do *not* have the
"ignore_resource_deletion" server feature, the behavior will change: we
will now ignore all data errors (including resource deletions) by default.
If the client wants to instead drop existing cached resources and start
failing data plane RPCs on data errors, they may update their bootstrap
to use the new "fail_on_data_errors" server feature instead.

This proposal removes the requirement from [gRFC A53] to log when
ignoring a resource deletion and then again when the resource comes back
into existence.  Instead, we will reflect ignored resource deletion via
both CSDS (see [gRFC A40]) and XdsClient metrics (see [gRFC A78]) by
setting the cache entry state to `NOT_FOUND` without actually deleting
the cached resource.

### Cache Entry States, CSDS, and XdsClient Metrics

The cache entry in the XdsClient is used to expose the client's state
via both CSDS ([gRFC A40]) and XdsClient metrics ([gRFC A78]).
Implementations must ensure that the data exposed to watchers is
consistent with the cache, so that CSDS and our metrics accurately
represent the resources used by the client.

Cache entries will contain both the currently used resource, if any, and
the most recent error seen, if any.  A cache entry must be in one of the
following states, the last two of which are new:

- `REQUESTED`: The client has subscribed to the resource but has not yet
  received the resource.  In this state, neither the resource nor the
  error will be present in the cache entry.
- `DOES_NOT_EXIST`: The resource does not exist, either because the
  resource timer has fired and the "resource_timer_is_transient_failure"
  server feature is not present, or because of an LDS or CDS resource
  deletion.  The entry will contain an error indicating how the deletion
  occurred.  If the "fail_on_data_errors" server feature is not present,
  the most recently seen resource may still be present in the cache entry.
- `ACKED`: The resource exists and is valid.  In this case, the error
  will be unset, and the resource will be present.
- `NACKED`: The most recent version of the resource seen from the
  control plane was invalid.  The error will indicate the validation
  failure.  If the "fail_on_data_errors" server feature is not present,
  the most recently seen resource may still be present in the cache entry.
- `RECEIVED_ERROR`: The client received an [xRFC TP3] error from the
  control plane.  The error will be the error received from the control
  plane.  If the "fail_on_data_errors" server feature is not present, the
  most recently seen resource may still be present in the cache entry.
- `TIMEOUT`: The client has subscribed to the resource but did not
  receive it before the timer fired, and the
  "resource_timer_is_transient_failure" server feature is present.  The
  resource will be unset and the error will indicate the timeout.

Those states map directly to the states in the CSDS
`ClientResourceStatus` enum.

[gRFC A78] defines a set of XdsClient metrics to be exported via
OpenTelemetry.  One of the metric labels defined there is the
`grpc.xds.cache_state` label, which is used for the
`grpc.xds_client.resources` metric.  In keeping with the new cache
states defined here, we will add the following new possible values for
this label:
- `timeout`: The cache entry is in TIMEOUT state.
- `does_not_exist_but_cached`: The cache entry is in DOES_NOT_EXIST
  state but does contain a cached resource (i.e., we have seen an LDS or
  CDS resource deletion but have ignored it).
- `received_error`: The cache entry is in RECEIVED_ERROR state, and no
  resource is cached.
- `received_error_but_cached`: The cache entry is in RECEIVED_ERROR
  state, and there is a cached resource (i.e., we have ignored a data
  error due to "fail_on_data_errors" not being present).

### Summary of Behavior Changes

The following table shows the old and new behavior for each case.

Error | Resource Already Cached | fail_on_data_errors Server Feature | Old Behavior | New Behavior
----- | ----------------------- | ---------------------------------- | ------------ | ------------
xDS channel reports TRANSIENT_FAILURE | false | (any) | `OnError()`, Fail data plane RPCs | `OnResourceChanged(Status)`, Fail data plane RPCs
xDS channel reports TRANSIENT_FAILURE | true | (any) | `OnError()`, Ignore | `OnAmbientError(Status)`, Ignore
ADS stream failed without reading a response | false | (any) | `OnError()`, Fail data plane RPCs | `OnResourceChanged(Status)`, Fail data plane RPCs
ADS stream failed without reading a response | true | (any) | `OnError()`, Ignore | `OnAmbientError(Status)`, Ignore
NACK from client | false | (any) | `OnError(status)`, Fail data plane RPCs | `OnResourceChanged(Status)`, Fail data plane RPCs
NACK from client | true | false | `OnError(status)`, Ignore | `OnAmbientError(Status)`, Ignore
NACK from client | true | true  | `OnError(status)`, Ignore | `OnResourceChanged(Status)`, Drop existing resource and fail RPCs
Resource timeout | false | (any) | `OnResourceDoesNotExist()`, Fail data plane RPCs | `OnResourceChanged(Status(NOT_FOUND))` (status will be UNAVAILABLE if `resource_timer_is_transient_error` server feature is present), Fail data plane RPCs
LDS or CDS resource deletion from server | true | false | `OnResourceDoesNotExist()`, Drop existing resource and fail RPCs (watcher notification skipped if "ignore_resource_deletion" server feature is present) | `OnAmbientError(Status(NOT_FOUND))`, Ignore
LDS or CDS resource deletion from server | true | true | `OnResourceDoesNotExist()`, Drop existing resources and fail RPCs (watcher notification skipped if "ignore_resource_deletion" server feature is present) | `OnResourceChanged(Status(NOT_FOUND))`, Drop existing resource and fail RPCs
[xRFC TP3] error with status NOT_FOUND or PERMISSION_DENIED | false | (any) | N/A | `OnResourceChanged(status)`, Fail data plane RPCs
[xRFC TP3] error with status NOT_FOUND or PERMISSION_DENIED | true | false | N/A | `OnAmbientError(status)`, Ignore
[xRFC TP3] error with status NOT_FOUND or PERMISSION_DENIED | true | true | N/A | `OnResourceChanged(status)`, Drop existing resource and fail RPCs
[xRFC TP3] error with other status | false | (any) | N/A | `OnResourceChanged(status)`, Fail data plane RPCs
[xRFC TP3] error with other status | true | (any) | N/A | `OnAmbientError(status)`, Ignore

### Handling Ambient Errors

As mentioned above, the purpose of ambient errors is to provide useful
context in error messages that are triggered for reasons not directly
related to the ambient error.  For example, if we lose contact with the
xDS server, we continue to use cached resources, but if we subsequently
lose contact with all endpoints, we will fail data plane RPCs.  The RPC
failure is triggered by the fact that we've lost contact with all
endpoints, but the status message should include the fact that we've
lost contact with the xDS server, since the outage may be caused by the
fact that we haven't been getting EDS updates from the xDS server.

To support this, we will make two changes.

First, we will change pick_first and the petiole policies (see [gRFC A61])
to include the `resolution_note` field in all error messages generated
by those policies.

Second, we need to pass down the ambient errors in the `resolution_note`
when using xDS.  As per [gRFC A74], the `resolution_note` is passed down
as part of the `XdsConfig` struct generated by the XdsDependencyManager.
Note that the `resolution_note` field is in the nested `EndpointConfig`
struct.  There is a separate instance of that struct for each EDS
or LOGICAL_DNS cluster, which will be used by the branch of the LB
policy tree for that cluster.  Therefore, in order to get the right
set of ambient errors for each branch of the LB policy tree, the
`resolution_note` for each cluster will contain the ambient errors for
all resources for that cluster: ambient errors for the top-level LDS
and RDS resources will be included for all clusters, but each cluster
will include the ambient errors for only its own CDS and EDS resources.
Note that the same is true for LOGICAL_DNS clusters, except that there
are never any ambient errors from the EDS resource in that case.

If none of the resources for a given cluster have ambient errors, the
XdsDependencyManager will set the resolution note to indicate the xDS
node ID, which can be useful for debugging.

### Propagating Status to Data Plane RPCs

As noted in [gRFC A54], the set of status codes that gRPC should generate
for RPCs is intentionally limited.  In particular, INVALID_ARGUMENT and
NOT_FOUND are two of the codes that are reserved for application use
and should never be generated by gRPC itself.  Implementors should not
blindly propagate status codes from the XdsClient watcher to data plane
RPCs; instead, they should be careful to set status codes carefully on
data plane RPCs on both the client and server side.  In most cases, it
is expected that UNAVAILABLE is an appropriate code for data plane RPCs.

### Temporary environment variable protection

The new functionality will be guarded by the
`GRPC_EXPERIMENTAL_XDS_DATA_ERROR_HANDLING` env var until it passes
interop testing.  This env var will guard using the new response fields
from [xRFC TP3] and the new server features "fail_on_data_errors" and
"resource_timer_is_transient_error".  It will also guard whether we
default to ignoring resource deletion.

## Rationale

We considered an approach whereby we'd notify the watchers about whether
the error is a data error or a transient error and let the watcher make
the decision about how to respond.  However, that would make the watcher
API much harder to understand.  It would also break CSDS and XdsClient
metrics in that it would divorce the cache status inside the XdsClient
from the watcher behavior outside of the XdsClient.

## Implementation

C-core implementation:
- XdsClient watcher API change: https://github.com/grpc/grpc/pull/38269
- resolution_note handling: https://github.com/grpc/grpc/pull/38341 and
  https://github.com/grpc/grpc/pull/38411
- Support for "fail_on_data_errors": https://github.com/grpc/grpc/pull/38278
- Support for "resource_timer_is_transient_failure": https://github.com/grpc/grpc/pull/38377
- xRFC TP3 support: https://github.com/grpc/grpc/pull/38381

Will also be implemented in Java, Go, and Node.

## Open issues

To avoid problems where newly started clients fail when the control plane
is down, we have designed xDS fallback functionality, as described in
[gRFC A71](A71-xds-fallback.md).  However, because we do not want to
trigger fallback due to an individual resource not existing if the control
plane is actually responding, we do not currently trigger fallback when
the resource timer fires if we are still connected to the control plane.
This has led to concerns about failure modes where control plane slowness
may cause clients to act as if resources do not exist rather than using a
fallback server.  Ideally, it seems like that we would like to use a timer
to consider the xDS server connection to have failed if we haven't gotten
back a response in a certain amount of time, but we cannot do that while
we are depending on the timer to detect individual resources not existing.

As future work, we should consider whether there are ways to safely
trigger xDS fallback if the primary server is responding too slowly.
There are a number of challenges here:

- Given the number of issues we've seen where the control plane has been
  slow to respond, it's not clear to us that we would want all of those
  cases to trigger fallback, since we'd likely hit fallback way too often.
  We suspect that we would first need the control plane to monitor and
  provide an SLO for response time upon initial subscription to a resource.

- The current timer semantics are that we start a separate timer for each
  individual resource when it is initially subscribed to.  This ensures
  that any control plane slowness that affects the client's functionality
  at any time during the stream will cause us to see a problem.
  However, given that our criteria for switching back to the primary
  server is when we get a single resource back from the server, there
  is an incompatibility here: if the control plane is sending back at
  least one resource but is timing out for others, then we will have
  simultaneously met the criteria for using fallback and the criteria
  for switching back to the primary server.

- Given the above, it might seem beneficial to consider changing the timer
  semantics to just check for a timeout on any xDS response, rather
  than having a separate timer per resource.  Unfortunately, this won't
  actually work, because the xDS protocol does not require any minimum
  interval between responses.  Once the server has sent all of the
  resources to the client, if those resources never change, then the
  server never needs to send another response, and we would not want to
  trigger fallback in this case.

- As a middle ground, we could consider only having a timeout on the very
  first response on the ADS stream to trigger fallback, or even the
  very first response for each resource type.  However, this would
  still leave us blind to cases where the control plane is slow when new
  subscriptions are added later in the stream's lifetime, which we have
  seen in the wild.

- Even if we could identify the right timeout criteria to trigger fallback,
  we would need to decide whether or not to close the ADS stream to
  the primary, and both options here are problematic.  If we don't
  close the stream to the primary while the primary is timing out for
  some resources but still sending updates for others, then it's not
  clear how we decide when to switch back to the primary.  However,
  if we do close the stream and retry, and the new stream gets back at
  least one response before hitting another timeout, then we will wind
  up continually flapping between the primary and fallbacks servers,
  thus DoSing the fallback server with short-lived connections.

- Note: There's a pre-existing case here where the primary server
  can send an initial response and then the stream can die, and if that
  happens repeatedly over and over again, it will never cause us to
  trigger fallback, because we don't consider the stream to have failed
  if we got a response on it before it died.  That also means that we
  don't impose exponential backoff between stream attempts, so we could
  DoS the server.  But that problem already exists and is not new here.

It's not at all clear what the actual requirements are here, nor is it
clear what timer-based behavior can actually be safely used to trigger
xDS fallback, so we'll need a lot more discussion to come to consensus
on what we need here.  As a result, we are proposing that we proceed
with only the proposal above and handle this potential improvement as
a subsequent effort.
