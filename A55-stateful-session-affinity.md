A55: Stateful Session Affinity for Proxyless gRPC
----
* Author(s): Sanjay Pujare (@sanjaypujare)
* Approver: 
* Status: In Review
* Implemented in: <language, ...>
* Last updated: 2022-08-14
* Discussion at: https://groups.google.com/g/grpc-io/c/F84rRVNgml4

## Abstract

This design specifies a mechanism to implement session affinity where backend
services maintain a local state per "session" and RPCs of a session are always
routed to the same backend (as long as it is in a valid state) that is
assigned to that session. This specification is based on and is compatible
with [stateful session persistence][envoy-ssp] that is implemented in Envoy.
See Envoy PRs [17848][], [18207][].


## Background

Some users of proxyless gRPC need to use "stateful session affinity" in gRPC.
In this usage scenario, backend services maintain local state for RPC sessions
and once a particular "session" is assigned to a particular backend (by the
load-balancer), all future RPCs of that session must be routed to the same
backend.

This can be achieved by using HTTP cookies in the following manner.
A gRPC client application sends the first RPC in a session without
a cookie in the RPC. gRPC routes that RPC to some backend B1 based on the
configured LB policy. When gRPC receives the response for that first RPC from
B1, it attaches a configured cookie to the response before delivering the
response to the client application. The cookie encodes some identifying
information about B1 (for example its IP-address). The client application
saves the cookie and uses that cookie in all future RPCs of that session. gRPC
uses the cookie and gets the encoded backend information to route the RPC to
the same backend B1 if it is still in a valid state.

### Related Proposals: 

* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [A42: xDS Ring Hash LB Policy][A42]

## Proposal

This design involves the following 4 parts.

### Listener and Route Configuration through xDS

The feature is enabled and configured using the
[cookie based stateful session extension][cookie-ext] which
is an [http-filter][] in the HttpConnectionManager for the target service. A
sample configuration (expressed as yaml) is as follows:

```yaml
   http_filters:
   - name: envoy.filters.http.stateful_session
     typed_config:
       "@type": type.googleapis.com/envoy.extensions.filters.http.stateful_session.v3.StatefulSession
       session_state:
         name: envoy.http.stateful_session.cookie
         typed_config:
           "@type": type.googleapis.com/envoy.extensions.http.stateful_session.cookie.v3.CookieBasedSessionState
           cookie:
             name: global-session-cookie
             path: /Package1.Service2/Method3
             ttl: 120s
```

`name` is the name of the cookie used in this feature.

`path` is the path for which this cookie is applied.

`ttl` is the time-to-live for the cookie. It is set on the cookie but will not
be enforced by gRPC.

This new filter will be added to the HTTP Filter Registry and processed as
described in [A39: xDS HTTP Filter Support][A39]. Specifically when this filter
is present in the configuration, gRPC will create the appropriate client filter
(aka interceptor in Java/Go) and install it in the channel to process data
plane RPCs. We call this filter `CookieBasedStatefulSessionFilter`. It will
copy the 3 configuration values - `name`, `ttl`  and `path` -  into the filter
object. Validation and processing are further described in
[Stateful session][stateful-session]. Note that
[StatefulSessionPerRoute][stateful-session-per-route] *will* be supported.

The filter is described in more detail in the section
[`CookieBasedStatefulSessionFilter` Processing][filter-section]

### Load Balancer Configuration Containing `override_host_status`

Even with stateful session affinity enabled, gRPC will only send an RPC to a
valid backend (host) as defined by
[`common_lb_config.override_host_status`][or-host-status]. When the backend
state is invalid, gRPC will use the configured load balancing policy to pick
the backend.

Note that gRPC currently ignores endpoints with health status other than
`HEALTHY` or `UNKNOWN` and even `XdsClient` excludes such endpoints in its
update to the watchers. We will need a separate design to include such
endpoints if we determine that such behavior is needed for this feature to
be useful to our users.

As a result, for this design gRPC will only consider the `UNKNOWN` or
`HEALTHY` states that are specified in
[`common_lb_config.override_host_status`][or-host-status] and ignore all
other values. We considered the alternative of NACKing (i.e. rejecting)
the CDS update when unsupported values are present, however it was discarded
because of the difficulty in using such configuration in mixed deployments.

The `override_host_status` value is included in the config update sent to
the `“cluster_resolver_experimental”` policy which passes down the update to
the `“xds_cluster_impl”` policy where it is used to create the additional
child policy as described below.

### `CookieBasedStatefulSessionFilter` Processing

A channel configured for stateful session affinity will have the
`CookieBasedStatefulSessionFilter` installed (interceptor in Java/Go).
This filter will process incoming RPCs and set an appropriate LB pick value
which will be passed to the Load Balancing Policy's Pick method.

When an RPC arrives on a configured channel the Config Selector processes it
as described in [A31: gRPC xDS Config Selector Design][A31]. This includes the
`CookieBasedStatefulSessionFilter` which is installed as one of the filters in
the channel. Note that the filter maintains a context for an RPC and processes
both the outgoing request and incoming response of the RPC within that context.
Also note that the filter is initialized with 3 configuration values - `name`,
`ttl`  and `path` as described above.

The filter performs the following steps:

* If the incoming RPC “path” does not match the `path` configuration value,
skip any further processing and just pass the RPC through (to the next
filter). For example if the `path` configuration value is `/Package1.Service2`
and the RPC method is `/Package2.Service1/Method3` then just pass the RPC
through.

* Search the RPC headers (metadata) for header(s) named `Cookie` and get the
set of all the cookies present. If the set does not contain a cookie whose
name matches the `name` configuration value of the filter, then pass the RPC
through. If you find more than one cookie with that `name` then log a local
warning and pass the RPC through. Otherwise get the value of that cookie and
base64-decode the value to get the `override-host` for the RPC. This value
should be a syntactically valid IP:port address: if not, then log a local
warning and pass the RPC through. For a valid `override-host` value set it
in the RPC context as the value of `upstreamAddress` state variable which
will be used in the response processing of that RPC.

* In case of a valid `override-host` value, pass this value as a
"pick-argument" to the Load Balancer's pick method as described in [A31][].
For example,

    * in Java use `CallOptions` to pass this value.

    * in Go pass it via `Context`.

    * In C-core, this will be via the `GRPC_CONTEXT_SERVICE_CONFIG_CALL_DATA`
      call context element whose API will be extended to allow the
      filter to add a new call element.

* In the response path of the RPC, if the RPC path does not match the
  `path` config value of the filter then skip further processing and
  pass the response through. Otherwise check if `upstreamAddress` is
  set (in the RPC context of this filter) and get the value. Also get
  the peer address for the RPC; let’s call it `hostAddress` here. For
  example, in Java you get the peer address through the
  `Grpc.TRANSPORT_ATTR_REMOTE_ADDR` attribute of the current
  ClientCall. If `upstreamAddress` is not set or `upstreamAddress` and
  `hostAddress` are different, then create a cookie using our Cookie config
  which has the `name`, `ttl` and `path` values. Set the value of the cookie
  to be the base64-encoded string value of `hostAddress`. Add this Cookie to
  the response headers. The logic in pseudo-code is:

```C++
  if (!upstreamAddress.has_value() || hostAddress != upstreamAddress) {
    String encoded_address = base64_encode(hostAddress);
    Create a Cookie from our cookie configuration;
    Set cookie's value as encoded_address;
    Add the cookie header to the response;
  }
```

### LB Policy for Stateful Session Affinity

After the `CookieBasedStatefulSessionFilter` has passed the `override-host`
value to the Load Balancer as a "pick-argument", the Load Balancer uses
this value to route the RPC appropriately. The required logic needs to be
implemented in a new and separate LB policy at an appropriate place in the
hierarchy.

RPC load reporting happens in the `“xds_cluster_impl”` policy and we do want
all RPCs to be included in the load reports. And this policy conditionally
routes an RPC based on the `override-host` value if present, or else delegates
to the currently configured LB policy. Hence the new policy needs to be just
below the `“xds_cluster_impl”` as its child policy. Let's call this new policy
`“override-host-experimental”`. This policy has the required subchannel and
RPC routing logic as follows:

* maintain a map of addresses (IP:port) to subchannels - let’s call this map
  `overrideHostMap`.

* whenever a new subchannel is created (as a result of an endpoint update from
  the control plane), add a new entry to `overrideHostMap` with the subchannel
  address(es) as the key and the subchannel as the value.

    * In Java, this may result into more than one entry if the
      createSubchannel arg’s `equivalentGroupAddress` has more than one
      address.

    * in C-core and Node, LB policies create a new subchannel for every address
      every time there is a new address list. In the map, we will just replace
      an existing entry with the most recent subchannel created for a given
      address i.e. the new subchannel created will replace the older one for
      that address.

* whenever a subchannel is shut down remove the associated entries from
  `overrideHostMap`.

    * this can be achieved by wrapping a subchannel to intercept subchannel
      shutdown and the wrapper is returned by `createSubchannel`.

* the policy's subchannel picker works as follows:

    * when called to pick a subchannel, it gets the value of `override-host`
      that was passed from `CookieBasedStatefulSessionFilter` (as described
      above).

    * if the value is not present then just delegate to the child policy.

    * if the value is present, then get the address in the value to look up
      an entry in `overrideHostMap`. If no entry is found, then delegate to
      the child policy.

    * if an entry is found, get that subchannel’s connectivity state and
      if it is not one of [`CONNECTING`, `READY`, `IDLE`] then delegate to
      the child policy.

    * otherwise return the subchannel from the entry as the pick result.

Note that we create the `“override-host-experimental”` policy as the child of
`“xds_cluster_impl”` only when the `override_host_status` config is present
which is passed through
[`common_lb_config.override_host_status`][or-host-status]. The
`“cluster_resolver_experimental”` policy will receive this value in its CDS
update and pass this config via its child (`“priority_experimental”`) to the
`“xds_cluster_impl”` policy. Only when `override_host_status` is set it will
create the `“override-host-experimental”` policy as its child and make one of
the existing policies (`"wrr-locality-experimental"`, `"weighted_target"`
or `"ring_hash_experimental"`) as the child policy of
`“override-host-experimental”` as shown in the following diagram.

![LB Policy Hierarchy Diagram](A55_graphics/lb-hierarchy.png)


## Rationale

The stateful session affinity requirement is not satisfied by the
[ring-hash LB policy][A55] because any time a backend host is removed or
added, the hash mapping of 1/N client sessions gets affected (where N is the
total number of backends) which would lead to dropped requests in some cases
which is unacceptable.

This feature avoids the problem by using HTTP cookies as described above.

## Implementation

The design will be implemented in gRPC C++, Java and Go - in that order.
Wrapped languages should be able to use the feature provided they can do
cookie management in their client code.

Cookies are rarely used with gRPC so the implementations should also
include a recipe for cookie management. This could involve illustrative
examples using cookie jar implementations which are available in various
languages such as Java, Go and Node.js. Wherever possible the
implementations may also include a reference implementation of an interceptor
that does cookie management using a cookie jar object. The interceptor
is instantiated for a session and it processes the `"Set-Cookie"` header
in a response to save the cookie, and uses that cookie in subsequent
requests.


[envoy-ssp]: https://docs.google.com/document/d/1IU4b76AgOXijNa4sew1gfBfSiOMbZNiEt5Dhis8QpYg/edit#heading=h.sobqsca7i45e
[17848]: https://github.com/envoyproxy/envoy/pull/17848
[18207]: https://github.com/envoyproxy/envoy/pull/18207
[A27]: https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md
[A39]: https://github.com/grpc/proposal/blob/master/A39-xds-http-filters.md
[A42]: https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md
[A31]: https://github.com/grpc/proposal/blob/master/A31-xds-timeout-support-and-config-selector.md#new-functionality-in-grpc
[cookie-ext]: https://www.envoyproxy.io/docs/envoy/v1.22.0/api-v3/extensions/http/stateful_session/cookie/v3/cookie.proto.html
[http-filter]: https://github.com/grpc/proposal/blob/master/A39-xds-http-filters.md
[stateful-session]: https://www.envoyproxy.io/docs/envoy/v1.22.0/configuration/http/http_filters/stateful_session_filter#config-http-filters-stateful-session
[stateful-session-per-route]: https://www.envoyproxy.io/docs/envoy/v1.22.0/api-v3/extensions/filters/http/stateful_session/v3/stateful_session.proto#extensions-filters-http-stateful-session-v3-statefulsessionperroute
[or-host-status]: https://github.com/envoyproxy/envoy/blob/15d8b93608bc5e28569f8b042ae666a5b09b87e9/api/envoy/config/cluster/v3/cluster.proto#L615
[filter-section]: #cookiebasedstatefulsessionfilter-processing
[rfc-6265]: https://www.rfc-editor.org/rfc/rfc6265.html
