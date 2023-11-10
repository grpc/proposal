Node: Server API to drain connections on a port
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: In Review
* Implemented in: Node.js
* Last updated: 2023-11-09
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add a new method `bind` to the `Server` class to close all active connections on a specific port or all ports

## Background

[gRFC A36: xDS-Enabled Servers][A36] specifies that an update to a Listener resource causes all older connections on that Listener to be gracefully shut down with a grace period for long-lived RPCs.


### Related Proposals:
* [gRFC A36: xDS-Enabled Servers][A36]
* [gRFC L109: Node: Server API to unbind ports][L109]

## Proposal

A single method `drain(port: string, graceTimeMs: number): void` will be added to the `Server` class. When called, all open connections associated with the specified `port` will be closed gracefully. After `graceTimeMs` milliseconds, all open streams on those connections will be cancelled.

The `port` string will be handled the same as in [gRFC L109][L109]: the name will be normalized, and then used to reference the existing list of bound ports. Port 0 is handled specially: a call to `drain` with port 0 will throw an error. After a call to `bindAsync` with a port number of 0 succeeds, and the specific bound port number is provided in the callback, `drain` can be called with the original `port` string with the bound port number substituted for port 0 to drain connections on that port.

## Rationale

### Alternatives considered

#### Make relevant internals `protected`

When implementing [gRFC A36][A36], I intend to make the xDS server a subclass of the `Server` class. TypeScript supports using the `protected` keyword to make fields visible to subclasses. So, it would be possible to make the relevant internals `protected` to be able to implement this functionality directly in the xDS server class. However, doing so makes those fields visible to any user that wants to subclass `Server`, effectively making them part of the library's public API. That would put significant limits on how those fields could change in the future, which I am not currently willing to accept.

#### Make the arguments optional

The specific use case described in [gRFC A36][A36] only calls for draining a specific port, and the grace period is always applied, so the API defined here fully handles that use case. The arguments could be made optional

## Implementation

I (murgatroid99) will implement this.

[A36]: https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md
[L109]: https://github.com/grpc/proposal/blob/master/L109-node-server-unbind.md
