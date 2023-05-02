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

When connections to all addresses fail, there are some similarities and some
differences between the Java/Go implementations and the C-core implementation.

Similarities include:
- Reporting `TRANSIENT_FAILURE` as the connectivity state to the channel.
- Applying exponential backoff. During the backoff period, [wait_for_ready][wfr]
  are queued while other RPCs fail.

Differences show up after the backoff period ends:
- C-core remains in `TRANSIENT_FAILURE` while Java/Go move to `IDLE`.
- C-core attempts to reconnect to the given addresses, while Java/Go rely on the
  client application to make an RPC or an explicit attempt to connect.
- C-core moves to `READY` only when a connection attempt succeeds.

This behavior of staying in `TRANSIENT_FAILURE` until it can report `READY` is
called sticky TRANSIENT_FAILURE, or sticky-TF.

[wfr]: https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md

### Related Proposals:
* [gRFC A37: xDS Aggregate and Logical DNS Clusters][A37]
* [gRFC A56: `priority_experimental` LB policy][A56]

[A37]: A37-xds-aggregate-and-logical-dns-clusters.md
[A56]: A56-priority-lb-policy.md

## Proposal

Specific changes are described in their own subsections below.

### Use sticky-TF by default

Current `pick_first` implementations that don't provide sticky-TF have the
following shortcomings:
- When none of the received addresses are reachable, client applications
  experience long delays before their RPCs fail. This is because the channel
  does not spend enough time in `TRANSIENT_FAILURE` and goes back to
  `CONNECTING` state while attempting to reconnect.
- `priority` LB policy maintains an ordered list of child policies, and sends
  picks to the highest priority child reporting `READY` or `IDLE`. It expects
  child policies to support sticky-TF, and if not, it can result in picks being
  sent to a higher priority child with no reachable backends, instead of a lower
  priority child that is reporting `READY`. See [gRFC A56][A56] for more
  details.

One scenario where this comes up is when a `LOGICAL_DNS` cluster is used under
an aggregate cluster, and where the `LOGICAL_DNS` cluster is not the last
cluster in the list. Each cluster under the aggregate cluster is represented as
a child policy under `priority`, and the leaf policy for a `LOGICAL_DNS` cluster
is `pick_first`. Without sticky-TF support in `pick_first`, it can lead to a
situation where the `priority` LB policy continues to send picks to a higher
priority `LOGICAL_DNS` cluster when none of the addresses behind it are
reachable, because `pick_first` doesn't report `TRANSIENT_FAILURE` as it's
connectivity state. See [gRFC A37][A37] for more details on aggregate clusters.

Providing sticky-TF in all `pick_first` implementations would enable us to
overcome these shortcomings. This would involve making the following changes to
`pick_first` implementations, once connections to all addresses fail:
- Report `TRANSIENT_FAILURE` as the connectivity state.
- Attempt to reconnect to the addresses indefinitely until a connection succeeds
  (at which point, they should report `READY`), or there is no RPC activity on
  the channel for the specified `IDLE_TIMEOUT`.


All gRPC implementations should implement `IDLE_TIMEOUT` and have it enabled by
default. A default value of 30 minutes is recommended.

[A56]: A56-priority-lb-policy.md
[A37]: A37-xds-aggregate-and-logical-dns-clusters.md

### Enable random shuffling of address list

Our general philosophy is that the address order is to be determined by the name
resolver server, or by the name resolver client performing sorting as described
in [RFC-8304 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4). In
some DNS configurations though, all clients get the addresses in the same order,
either because the authoritative server does that or because of caching. In
these scenarios, it is desirable for `pick_first` to support random shuffling of
received addresses to provide better load balancing.


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

gRPC clients that receive `pick_first` configuration with this field set, but do
not support this configuration should continue to work and their behavior should
remain unchanged. Implementations that support the new field and receive
configuration with `shuffleAddressList` set to `true`, should shuffle the
received address list at random before attempting to connect to them.

### Temporary environment variable protection

During initial development, random shuffling of address list will be enabled by
the `GRPC_EXPERIMENTAL_PICKFIRST_LB_CONFIG` environment variable.

## Rationale

N/A


## Implementation

Will be implemented in C-core, Java, Go, and Node.

## Open issues (if applicable)

N/A
