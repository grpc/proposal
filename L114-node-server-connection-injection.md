L114: Node Server Connection Injection
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: In Review
* Implemented in: Node.js
* Last updated: 2024-02-09
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add a new method `createConnectionInjector` to the `Server` class to allow existing TCP connections or TCP connection-like objects to be injected into the server. These connections would have the TLS handshake conducted according to the credentials provided in the call to that method, and then method handlers and interceptors for the server would apply as usual.

## Background

As a part of the design [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29], a single server can apply different security configurations to different incoming connections on the same port depending on properties of those connections. This is not possible with the existing Node gRPC Server implementation, because the Node HTTP2 server listens on a port and automatically performs the TLS handshake for every incoming connection.

In addition, this functionality has been requested in [grpc/grpc-node#2317](https://github.com/grpc/grpc-node/issues/2317).

### Related Proposals:
* [A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [L111: Node: Server API to drain connections on a port][L111]
* [L109: Node: Server API to unbind ports][L109]

## Proposal

We will add a new method `createConnectionInjector(credentials: ServerCredentials): ConnectionInjector` to the `Server` class. The `ConnectionInjector` class has the following API:

```ts
interface ConnectionInjector {
  injectConnection(connection: stream.Duplex): void;
  drain(graceTimeMs: number): void;
  destroy(): void;
}
```

The `injectConnection` method accepts any duplex byte stream object, represented as the built in `stream.Duplex` class. The built in APIs represent TCP connections with the `net.Socket` class, which is a subclass of `stream.Duplex`. The server will perform the TLS handshake with the specified credentials and then handle the connection just like any other connection that comes in on a listening port.

The `drain` method gracefully closes all open connections injected in to this `ConnectionInjector`, similar to the `Server#drain` method defined in [gRFC L111][L111].

The `destroy` method shuts down the `ConnectionInjector` and gracefully closes all open connections injected into it, similar to the `Server#unbind` method defined in [gRFC L109][L109].

## Rationale

### `drain` method

The xDS Server needs to be able to drain existing connections after receiving an update to the `Listener` resource. A `drain` method on the `ConnectionInjector` provides a simple way to do that and it matches an existing `Server` method.

### `destroy` method

A connection injector does not own any listening TCP ports, so it generally does not represent resources that need to be released. However, [gRFC A29][A29] introduces new credentials types that are more resource-intensive, so it is useful to be able to release references to those.

### Alternatives considered

#### Opaque handle usable in existing APIs

An alternative design is for `createConnectionInjector` to return an opaque object (`Handle`) that can be passed as an argument to the `drain` and `unbind` methods, in addition to another new `Server` method `injectConnection(handle: Handle, connection: stream.Duplex)`. This is functionally equivalent to the proposed design, but I think it's cleaner to have that functionality in an object.

## Implementation

I (murgatroid99) will implement this in parallel with the design review.

[A29]: https://github.com/grpc/proposal/blob/master/A29-xds-tls-security.md
[L111]: https://github.com/grpc/proposal/blob/master/L111-node-server-drain.md
[L109]: https://github.com/grpc/proposal/blob/master/L109-node-server-unbind.md
