A62: `pick_first` LB policy
----
* Author(s): Easwar Swaminathan (@easwars)
* Approver: @markdroth
* Status: Draft
* Implemented in:
* Last updated: 2023-04-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This design specifies a configuration option for the `pick_first` LB policy that
would enable it to randomly shuffle the order of the addresses it receives
before it starts to attempt to connect.

Given that `pick_first` is the default LB policy in all gRPC implementations, it
is necessary to have a written specification that details its behavior. This
document provides that specification.

## Background

All gRPC implementations contain a simple load balancing policy named
`pick_first` that can be summarized as follows:
- It takes a list of addresses from the name resolver and attempts to connect to
  those addresses one at a time, in order, until it finds one that is reachable.
  All RPCs sent on the gRPC channel will be sent to this address.
- If this connection breaks at a later point in time, pick_first will not
  attempt to reconnect until the application requests that it does so.
- If none of the addresses are reachable, it applies an exponential backoff
  before attempting to reconnect.

Since `pick_first` was implemented prior to the first stable version of any gRPC
language, no gRFC exists for it. Although it is a simple LB policy whose
behavior can be summarized in a few sentences, as done above, there are some
subtle differences in implementation across the various languages. This document
aims to provide a more detailed description of its behavior, with the aim of
bringing convergence among the different implementations.

## Proposal

Specific scenarios are described in their own subsections below.

### When connections to all addresses fail

`pick_first` LB policy attempts to connect to the given addresses in order, and
when none of the addresses are reachable, it must:
- report `TransientFailure` as the connectivity state to the channel
- apply exponential backoff (all RPCs attempted during this period will fail)
- attempt to reconnect to the given addresses in order
- continue to report `TransientFailure` as the connectivity state, until a
  connection succeeds, at which point, it must report `Ready`

This behavior of staying in `TransientFailure` until it can report `Ready` is
called sticky-TransientFailure.

`pick_first` LB policy should indefinitely attempt to reconnect, in a bid to
move out of `TransientFailure` and into `Ready`, until the channel enters idle
mode due to inactivity, at which point, the channel will shut down the name
resolver and the LB policy. See [gRPC Connectivity Semantics](1) for more
details about idleness.

[1]: https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md

### Enable random shuffling of address list

At the time of this writing, `pick_first` implementations do not expect any
configuration to be passed to it. As part of this design, we will add a field to
its configuration.

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

### Temporary environment variable protection

During initial development, random shuffling of address list will be enabled by
the `GRPC_EXPERIMENTAL_PICKFIRST_LB_CONFIG` environment variable.

## Rationale

N/A


## Implementation

Will be implemented in C-core, Java, Go, and Node.

## Open issues (if applicable)

N/A
