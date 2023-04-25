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
- Applying exponential backoff. During the backoff period, `wait_for_ready` RPCs
  are queued while other RPCs fail.

Differences show up after the backoff period ends:
- C-core remains in `TRANSIENT_FAILURE` while Java/Go move to `IDLE`.
- C-core attempts to reconnect to the given addresses, while Java/Go rely on the
  client application to make an RPC or an explicit attempt to connect.
- C-core moves to `READY` only when a connection attempt succeeds.

This behavior of staying in `TRANSIENT_FAILURE` until it can report `READY` is
called sticky-TransientFailure, or sticky-TF.

## Proposal

Specific changes are described in their own subsections below.

### Support Sticky-TF

All `pick_first` implementations should support sticky-TF. Once connections to
all addresses fail, they should:
- Report `TRANSIENT_FAILURE` as the connectivity state.
- Attempt to reconnect to the addresses indefinitely until a connection succeeds
  (at which point, they should report `READY`), or the channel becomes idle (see
  [gRPC Connectivity Semantics](1) for more details about idleness).

Supporting sticky-TF has the following advantages:
- Avoids long delays before failing RPCs while the channel goes back to
  `CONNECTING` state while attempting to reconnect.
- Allows `pick_first` to work as a child of the [priority LB policy](2).

[1]: https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md
[2]: https://github.com/grpc/proposal/blob/master/A56-priority-lb-policy.md

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

gRPC clients that receive `pick_first` configuration with this field set, but do
not support this configuration should continue to work and their behavior should
remain unchanged. Implementations that support the new field should shuffle the
received address list at random before attempting to connect to them.

Our general philosophy is that the address order is to be determined by the name
resolver server, or by the name resolver client performing sorting as described
in [RFC-8304 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4). But
having this option in `pick_first` can be beneficial in some DNS configurations
where all clients get the addresses in the same order (e.g., either because the
authoritative server does that or because of caching) but where it is desirable
to randomize the order of the addresses to provide better load balancing.

### Temporary environment variable protection

During initial development, random shuffling of address list will be enabled by
the `GRPC_EXPERIMENTAL_PICKFIRST_LB_CONFIG` environment variable.

## Rationale

N/A


## Implementation

Will be implemented in C-core, Java, Go, and Node.

## Open issues (if applicable)

N/A
