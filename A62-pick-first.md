A62: `pick_first`: sticky TRANSIENT_FAILURE and address order randomization 
----
* Author(s): Easwar Swaminathan (@easwars)
* Approver: @markdroth
* Status: Draft
* Implemented in:
* Last updated: 2023-04-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This document describes a couple of changes being made to the `pick_first` LB
policy with regards to
- Expected behavior when connections to all addresses fail, and
- Support for address order randomization

## Background

All gRPC implementations contain a simple load balancing policy named
`pick_first` that can be summarized as follows:
- It takes a list of addresses from the name resolver and attempts to connect to
  those addresses one at a time, in order, until it finds one that is reachable.
- Once it finds a reachable address:
  - All RPCs sent on the gRPC channel will be sent to this address.
  - If this connection breaks at a later point in time, `pick_first` will not
    attempt to reconnect until the application requests that it does so, or makes
    an RPC.
- If none of the addresses are reachable, it applies an exponential backoff
  before attempting to reconnect.

There are a few problems with the existing `pick_first` functionality, which
will be described in the following subsections.

### Sticky Transient Failure

When connections to all addresses fail, there are some similarities and some
differences between the Java/Go implementations and the C-core implementation.

Similarities include:
- Reporting `TRANSIENT_FAILURE` as the connectivity state to the channel.
- Applying exponential backoff. During the backoff period, [wait_for_ready][wfr]
  RPCs are queued while other RPCs fail.

Differences show up after the backoff period ends:
- C-core remains in `TRANSIENT_FAILURE` while Java/Go move to `IDLE`.
- C-core attempts to reconnect to the given addresses, while Java/Go rely on the
  client application to make an RPC or an explicit attempt to connect.
- C-core moves to `READY` only when a connection attempt succeeds.

This behavior of staying in `TRANSIENT_FAILURE` until it can report `READY` is
called sticky TRANSIENT_FAILURE, or sticky-TF.

Current `pick_first` implementations that don't provide sticky-TF have the
following shortcomings:
- When none of the received addresses are reachable, client applications
  experience long delays before their RPCs fail. This is because the channel
  does not spend enough time in `TRANSIENT_FAILURE` and goes back to
  `CONNECTING` state while attempting to reconnect.
- [priority LB policy][A56] maintains an ordered list of child policies, and
  sends picks to the highest priority child reporting `READY` or `IDLE`. It
  expects child policies to support sticky-TF, and if not, it can result in
  picks being sent to a higher priority child with no reachable backends,
  instead of a lower priority child that is reporting `READY`. This comes up is
  in xDS in the following scenario:
    - A `LOGICAL_DNS` cluster is used under an aggregate cluster, and the
      `LOGICAL_DNS` cluster is not the last cluster in the list.
    - Each cluster under the aggregate cluster is represented as a child policy
      under `priority`, and the leaf policy for a `LOGICAL_DNS` cluster is
      `pick_first`.
    - Without sticky-TF support in `pick_first`, it can lead to a situation
      where the `priority` LB policy continues to send picks to a higher
      priority `LOGICAL_DNS` cluster when none of the addresses behind it are
      reachable, because `pick_first` doesn't report `TRANSIENT_FAILURE` as its
      connectivity state. See [gRFC A37][A37] for more details on aggregate
      clusters.

[wfr]: https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md

### L4 Load Balancing

Because `pick_first` sends all requests to the same address, it is often used
for L4 load balancing by randomizing the order of the addresses used by each
client.  In general, gRPC expects address ordering to be determined as part of
name resolution, not by the LB policy. For example, DNS servers may randomize
the order of addresses when there are multiple A/AAAA records, and the DNS
resolver in gRPC is expected to perform [RFC-6724][6724] address sorting.
However, there are some cases where DNS cannot randomize the address order,
either because the DNS server does not support that functionality or because it
is defeated by client-side DNS caching. To address such cases, it is desirable
to add a client-side mechanism for randomly shuffling the order of the
addresses.

[6274]: https://www.rfc-editor.org/rfc/rfc6724.html

### `pick_first` via xDS

There are cases where it is desirable to perform L4 load balancing using
`pick_first` when getting addresses via xDS instead of DNS. As a result, we need
a way to configure use of this LB policy via xDS.

Note that client-side address shuffling may be equally desirable in this case,
since the xDS server may send the same EDS resource (with the same endpoints in
the same order) to all clients.

### Related Proposals:

* [gRFC A37: xDS Aggregate and Logical DNS Clusters][A37]
* [gRFC A52: gRPC xDS Custom Load Balancer Configuration][A52]
* [gRFC A56: `priority_experimental` LB policy][A56]

[A37]: A37-xds-aggregate-and-logical-dns-clusters.md
[A52]: A52-xds-custom-lb-policies.md
[A56]: A56-priority-lb-policy.md

## Proposal

Specific changes are described in their own subsections below.

### Use sticky-TF by default

Using sticky-TF by default in all `pick_first` implementations would enable us
to overcome the shortcomings described [here](#sticky-transient-failure). This
would involve making the following changes to `pick_first` implementations, once
connections to all addresses fail:
- Report `TRANSIENT_FAILURE` as the connectivity state.
- Attempt to reconnect to the addresses indefinitely until a connection succeeds
  (at which point, they should report `READY`), or there is no RPC activity on
  the channel for the specified `IDLE_TIMEOUT`.

All gRPC implementations should implement `IDLE_TIMEOUT` and have it enabled by
default. A default value of 30 minutes is recommended.

### Enable random shuffling of address list

At the time of this writing, `pick_first` implementations do not expect any
configuration to be passed to it. As part of this design, we will add a field to
its configuration that would enable it to randomly shuffle the order of the
addresses it receives before it starts to attempt to connect.

```
{
    // If set to true, instructs the LB policy to shuffle the order of the
    // list of addresses received from the name resolver before attempting to
    // connect to them.
    "shuffleAddressList": boolean
}
```

gRPC clients that receive `pick_first` configuration with this field set, should
behave as follows:
- If they do not support this configuration, they should continue to work as
  before and their behavior should remain unchanged.
- If they support this configuration, they should shuffle the received address
  list at random before attempting to connect to them.

### `pick_first` via xDS

gRPC recently added support for custom load balancer configuration to be
specified by the xDS server. See [gRFC A52][A52] for more details.

To enable the xDS server to specify `pick_first` using this mechanism, an
extension configuration message was added as part of [Envoy PR
#26952](https://github.com/envoyproxy/envoy/pull/26952).

gRPC's Custom LB policy functionality will be enhanced to support this new
extension and will result in the `pick_first` LB policy being used as the
locality and endpoint picking policy.

### Temporary environment variable protection

During initial development, the `GRPC_EXPERIMENTAL_PICKFIRST_LB_CONFIG`
environment variable will guard the following:
- `shuffleAddressList` configuration knob in the `pick_first` LB policy
- Accepting the [PickFirst][pf_xds] config message as a [Custom LB policy][A52]
  in xDS

[pf_xds]: https://github.com/envoyproxy/envoy/blob/3ea7ff04dd421646f6154dd5d0f6bd0f241c5ce2/api/envoy/extensions/load_balancing_policies/pick_first/v3/pick_first.proto#L18

## Rationale

N/A

## Implementation

Will be implemented in C-core, Java, Go, and Node.

## Open issues (if applicable)

N/A
