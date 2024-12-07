A88: xDS Data Error Handling
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-12-06
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This document proposes adding support for the new xDS error propagation
mechanism described in [xRFC TP3], which provides a cleaner mechanism
for detecting non-existing resources.  It also defines improved behavior
for handling all xDS-related data errors.

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
  as described in [gRFC A57].
- **Data errors:** This includes things like resource validation failures
  that cause the client to NACK an update.  It also includes errors
  reported by the control plane via the new mechanism described in
  [xRFC TP3], including the case where a subscribed resource does not
  exist.

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

1. We can ignore the error and keep using the previously cached resource.
   This approach is safer from the perspective of helping to avoid an
   immediate outage.  However, it does rely on the control plane identifying
   the problem and alerting humans to take action, or else the problem
   becomes completely silent, which might simply defer the outage until
   some arbitrary amount of time later when the clients are restarted,
   at which point it will be harder to debug.

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
cached resource (i.e., approach 2 above), but we provide an option in
the bootstrap config to inhibit this behavior (see [gRFC A53]).  However,
all other data errors are actually treated the same as transient errors:
we unconditionally stick with the previously cached resource (i.e.,
approach 1 above).

This proposal changes the data error handling behavior to match the
principles outlined above.

### The Resource Timer

The xDS SotW protocol does not currently provide a way for servers to
explicitly inform clients when a subscribed resource does not exist,
so clients have to impose a 15-second does-not-exist timer for each
individual resource when the client initially subscribes to it, as
described in the [xDS protocol
documentation](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist).  The exact semantics of
this resource timer are tricky to get right, as described in [gRFC A57].[^1]

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
- [xRFC TP3: Error Propagation][xRFC TP3]

[gRFC A53]: A53-xds-ignore-resource-deletion.md
[gRFC A57]: A57-xds-client-failure-mode-behavior.md
[xRFC TP3]: https://github.com/cncf/xds/pull/107

## Proposal

This proposal has the following parts:
- changes to the resource timer behavior
- a mechanism to control data error handling behavior
- changes in the XdsClient watcher APIs
- support for errors provided by the xDS server as per [xRFC TP3]
- deprecating the "ignore_resource_deletion" server feature

### Changes to Resource Timer Behavior

If we know that the xDS server will use [xRFC TP3] to inform us of
non-existing resources, then we no longer need to rely on the resource
timer to detect non-existing resources, so we can instead re-purpose
the timer to detect slow xDS servers.

To that end, we will introduce a new server feature in the bootstrap
config called "resource_timer_is_transient_error".  When this server
feature is present, there will be two changes to the behavior of the
resource timer:
- The timer will be set for 30 seconds instead of 15 seconds.
- When the timer fires, it will be treated as a transient error
  instead of a data error.  (See below for how this is reflected in the
  watcher API.)

This server feature should be used only in cases where the server is known
to support [xRFC TP3].  If the server feature is used when the server
does *not* support [xRFC TP3], then the client will never report a
non-existant resource; it will instead treat a timeout as a transient
error talking to the xDS server.

Note that the client will honor [xRFC TP3] errors returned by the server
regardless of whether this server feature is present.  The presence of
this server feature actually controls only whether we can *depend* on the
server to provide these errors, so that we can be more lax about the
resource timer.

### Mechanism to Control Data Error Handling Behavior

As described above, the behavior that we want on data errors depends
on whether we can depend on the control plane to properly alert human
operators when these errors occur.  To configure this, we will introduce a
new server feature in the bootstrap config called "fail_on_data_errors",
which indicates that we cannot depend on the control plane to properly
alert human operators.

This server feature controls the behavior for handling data errors when
we already have a cached version of the resource.  By default, we will
ignore the error and continue using the cached version of the resource.
However, if this server feature is set, then we will instead discard any
previously cached version of the resource and start failing data plane
RPCs.

### Changes to XdsClient Watcher APIs

The current `XdsClient` API has two error-handling methods:

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

These methods do not map well to the two error categories defined above.

This proposal replaces those two methods with the following new methods,
which do map more cleanly to the two error categories defined above:

- `OnTransientError(Status status)`: Will be invoked for transient errors
  only.  This includes the ADS channel reporting TRANSIENT_FAILURE and
  the ADS stream terminating without receiving a response, as described
  in [gRFC A57].  It also includes the resource timer firing if the
  "resource_timer_is_transient_error" server feature is present.  The
  watcher is generally expected to ignore the error if it already has a
  valid cached resource.

- `OnClientDataError(Status status, bool fail_on_data_errors)`: Will be
  invoked for client-generated data errors only.  This is used when
  we NACK a resource.  The `fail_on_data_errors` parameter indicates
  whether the server feature of the same name was present in the
  bootstrap configuration for the xDS server that caused the data
  error.  If the watcher already has a valid resource, it will use the
  `fail_on_data_errors` parameter to determine whether it should continue
  using that resource or whether it should stop using that resource and
  put itself into a failing state.

- `OnServerDataError(Status status, bool fail_on_data_errors)`: Will be
  invoked for server-generated data errors only.  This is used for
  [xRFC TP3] errors and when the does-not-exist timer fires when the
  "resource_timer_is_transient_error" server feature is *not* present.
  The `fail_on_data_errors` parameter indicates whether the server feature
  of the same name was present in the bootstrap configuration for the
  xDS server that caused the data error.  If the watcher already has a
  valid resource, it will use the two parameters to determine whether it
  should continue using that resource or whether it should stop using
  that resource and put itself into a failing state.  See [xRFC TP3]
  for guidance on what status codes should be used to decide to drop an
  existing resource.

For transient connectivity errors, including the ADS channel reporting
TRANSIENT_FAILURE and the ADS stream terminating without receiving a
response (as described in [gRFC A57]), we will call `OnTransientError()`
instead of `OnError()`, but the semantics and behavior will remain
the same.  The watcher will ignore these errors if there is a previously
cached version of the resource.

For NACKs, we will call `OnClientDataError()`.  The watcher behavior will
depend on the presence of the "fail_on_data_errors" server feature in
the bootstrap configuration: if the server feature is present, we will
drop any previously cached resource and fail data plane RPCs; otherwise,
we will continue using the previously cached resource if we have one.

For the resource timer, the behavior will depend on the presence of the
"resource_timer_is_transient_error" server feature in the bootstrap
configuration.  If that server feature is present, then when the
timer fires, we will call `OnTransientError()` with a status code
of UNAVAILABLE.  If the server feature is *not* present, then when
the timer fires, we will call `OnServerDataError()` with a status code
of NOT_FOUND.  In either case, since we use this timer only when we do
not have a previously cached version of the resource, the result will
be the same: we will fail data plane RPCs.

For [xRFC TP3] errors, we will call `OnServerDataError()`.  The watcher
behavior will depend on the value of the "fail_on_data_errors" parameter
and the status code.  In gRPC, our watchers will throw away a previously
seen resource if `fail_on_data_errors` is true and the status code is
either NOT_FOUND or PERMISSION_DENIED; in any other case, we will retain
the previously seen resource, if any.

TODO: Do we need to batch updates to the watcher somehow for the case
where we've got a previous version of a resource cached plus an error?
Previously, that was okay, because we'd just deliver both updates to the
watcher, one at a time.  But if the watcher is going to decide to throw
away the cached resource when it sees the error, maybe we need to
deliver both at once, in case a new watcher starts when we've already
got both the cached resource and the error?

### Support for Errors Provided by the xDS server as per [xRFC TP3]

We will add support for the new response fields added in [xRFC TP3].
There will be no configuration telling gRPC to use these fields; we will
unconditionally use them if they are present.

When we receive an error for a given resource, we will cancel the resource
timer, if any, and report the error to the watchers' `OnServerDataError()`
methods.

In the XdsClient cache, when we receive an error for a resource, we will
record the error but still keep the previous version of the resource
cached.  We will record the cache status as `RECEIVED_ERROR`, which will
be reported in CSDS.

### Deprecating "ignore_resource_deletion" Server Feature

Because we now want the ability to control the behavior of all data
errors consistently rather than special-casing resource deletion, we
will deprecate the "ignore_resource_deletion" server feature that was
added in [gRFC A53].

For existing bootstrap configs that have the "ignore_resource_deletion"
server feature, gRPC will ignore that server feature, but the new
default behavior is to ignore all data errors, including resource
deletion, so they will still get the right behavior.

For existing bootstrap configs that do *not* have the
"ignore_resource_deletion" server feature, the behavior will change: we
will now ignore all data errors (including resource deletions) by default.
If the client wants to instead drop existing cached resources and start
failing data plane RPCs on data errors, they may update their bootstrap
to use the new "fail_on_data_errors" server feature instead.

### Summary of Behavior Changes

The following table shows the old and new behavior for each case.

Data Error | fail_on_data_errors Server Feature | resource_timer_is_transient_error Server Feature | Old Watcher Notification | Old Behavior | New Watcher Notification | New Behavior
---------- | ---------------------------------- | --------------------------------------------------------- | ------------------------ | ------------ | ------------------------ | ------------
NACK from client | false | | `OnError(status)` | Ignore | `OnClientDataError(status, false)` | Ignore if already have resource
NACK from client | true  | | `OnError(status)` | Ignore | `OnClientDataError(status, true)`  | Drop existing resource and fail RPCs
Resource timeout | false | false | `OnResourceDoesNotExist()` | Fail RPCs (no existing resource) | `OnClientDataError(Status(NOT_FOUND), false)` | Fail RPCs (no existing resource)
Resource timeout | true  | false | `OnResourceDoesNotExist()` | Fail RPCs (no existing resource) | `OnClientDataError(Status(NOT_FOUND), true)`  | Fail RPCs (no existing resource)
Resource timeout | false | true | `OnResourceDoesNotExist()` | Fail RPCs (no existing resource) | `OnTransientError(Status(NOT_FOUND), false)` | Fail RPCs (no existing resource)
Resource timeout | true  | true | `OnResourceDoesNotExist()` | Fail RPCs (no existing resource) | `OnTransientError(Status(NOT_FOUND), true)`  | Fail RPCs (no existing resource)
LDS or CDS resource deletion from server | false | | `OnResourceDoesNotExist()`, but skipped if "ignore_resource_deletion" server feature is present | Drop existing resource and fail RPCs | `OnServerDataError(status, false)` | Ignore
LDS or CDS resource deletion from server | true | | `OnResourceDoesNotExist()`, but skipped if "ignore_resource_deletion" server feature is present | Drop existing resource and fail RPCs | `OnServerDataError(status, false)` | Drop existing resource and fail RPCs
[xRFC TP3] error with status NOT_FOUND or PERMISSION_DENIED | false | | N/A | N/A | `OnServerDataError(status, false)` | Ignore
[xRFC TP3] error with status NOT_FOUND or PERMISSION_DENIED | true | | N/A | N/A | `OnServerDataError(status, true)` | Drop existing resource and fail RPCs
[xRFC TP3] error with other status | false | | N/A | N/A | `OnServerDataError(status, false)` | Ignore
[xRFC TP3] error with other status | true | | N/A | N/A | `OnServerDataError(status, true)` | Ignore

### Temporary environment variable protection

The new functionality will be guarded by the
`GRPC_EXPERIMENTAL_XDS_DATA_ERROR_HANDLING` env var until it passes
interop testing.  This env var will guard using the new response fields
from [xRFC TP3] and the new server features "fail_on_data_errors" and
"resource_timer_is_transient_error".

TODO: figure out how to handle legacy "ignore_resource_deletion" server
feature for env var

## Rationale

TODO: Anything we need to note here?

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

@markdroth will implement in C-core.

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
  slow to respond, it's not clear to me that we would want all of those
  cases to trigger fallback, since we'd likely hit fallback way too often.
  I suspect that we would first need the control plane to monitor and provide
  an SLO for response time upon initial subscription to a resource.

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
on what we need here.  As a result, I am proposing that we proceed
with only the proposal above and handle this potential improvement as
a subsequent effort.
