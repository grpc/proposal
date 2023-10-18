Node: Server API to unbind ports
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: In Review
* Implemented in: Node.js
* Last updated: 2023-10-18
* Discussion at: https://groups.google.com/g/grpc-io/c/0R8lD87XDo8

## Abstract

Add a new method `unbind` to the `Server` class to unbind a port previously bound by `bindAsync`.

## Background

[gRFC A36: xDS-Enabled Servers](https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md) specifies that ports must be in the "serving" or "not serving" state. In Node.js, we have decided that the best way to represent this is with the bound state of the port. Since an xDS-enabled server may receive a configuration update that causes it to transition from the "serving" to "not serving" state, it needs to be able to unbind a previously-bound port.

### Related Proposals:
* [gRFC A36: xDS-Enabled Servers](https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md)

## Proposal

A single method `unbind(port: string): void` will be added to the `Server` class. When called, if the `port` argument matches the port argument to a previous call to `bindAsync`, any ports bound by that `bindAsync` call will be unbound, and a GOAWAY will be sent to any active connections that were established using that port. Those connections will still be tracked for shutdown methods, meaning that `tryShutdown` will wait for them to finish, and `forceShutdown` will close them.

If a previous call to `bindAsync` is still pending, the `unbind` call will cancel it if possible, or close the ports after they have been opened otherwise. If `unbind` is called while `bindAsync` is pending, and then `bindAsync` is called again with the same `port` and identical `credentials` while the previous `bindAsync` call is still pending, the previous attempt will be un-cancelled if possible, and the callbacks for both `bindAsync` calls will be called when binding is complete. If instead `bindAsync` is called again with different credentials, it will start a new separate bind attempt.

Ephemeral ports (port 0) are handled differently as a special case. A call to `unbind` with a port number of 0 will throw an error. After a call to `bindAsync` with a port number of 0 succeeds, and the specific bound port number is provided in the callback, `unbind` can be called with the original `port` string with the bound port number substituted for port 0 to unbind that port.

## Rationale

### Alternatives considered

#### Use a different server object for each port

As an alternative to unbinding ports in an existing server, the xDS server could use a separate server object under the hood for each port it wants to bind. The upside of this option is that it does not require any API changes. However, there are a few downsides:

 - The xDS server would need its own tracking for registered services, and possibly other things in the future such as server interceptors, in order to pass each of those things along to each underlying server it creates.
 - Channelz statistics would be fragmented between the different server objects.


## Implementation

I (murgatroid99) will implement this initially as `_unbind`, which is considered non-public because of the underscore prefix. If/when this proposal is accepted, I will also add the non-underscore-prefixed method.
