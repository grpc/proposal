Transport Selection
----
* Author(s): dfawley
* Approver: roth, ejona86
* Status: Draft
* Implemented in: none
* Last updated: 2024-04-25
* Discussion at: TODO: TBD

## Abstract

A standardized method to dynamically choose between multiple transports.

## Background

gRPC predominately uses HTTP/2 over TCP to communicate between client and
server. In some implementations, there are existing methods for circumenting
this to instead use an in-process transport. There are also use cases that would
like to take advantage of alternate transports, e.g.
[QUIC+HTTP/3](https://github.com/grpc/proposal/blob/master/G2-http3-protocol.md)
and a way of communicating over shared memory (design TBD).

### Related Proposals: 

* [L37: Go Custom Transports](https://github.com/grpc/proposal/pull/103)
  (pending)
* [G2: gRPC over
  HTTP/3](https://github.com/grpc/proposal/blob/master/G2-http3-protocol.md)
* [A6: Client
  Retries](https://github.com/grpc/proposal/blob/master/A6-client-retries.md)

## Proposal

This proposal is for a standard way of selecting which transport to use in gRPC
clients and servers. The exact details will be left up to gRPC implementations
to decide; only the high-level method is included in this design.

The following are the high level components of this design:

* Client: Name Resolvers and LB policies specify an address _type_ for every
  address.

* Server: Users configure a listening address and corresponding address type.

* Client and Server: A registry is used to determine how to create transports
  for each address type.

* Client and Server: Define requirements for transport and stream APIs.  The
  specifics of the APIs will be left up to the individual gRPC client library
  authors.

### Client-Side

Addresses produced by a name resolver or provided to the channel by the LB
policy will contain an indicator of the address type. This could be a separate
field in a struct, or via the language's type system, or through encoding of the
address itself (e.g. using a URI like `"tcp:127.0.0.1:8080"`). A registry or
similar mechanism containing a mapping from address types to transport factories
will be used to determine how to create a transport for each address.

### Server-Side

Servers wishing to use a specific type of transport listener will specify the
address type along with the address to listen on. As with client addresses, this
could be a field in a struct, types, or by using a URI. Also like client address
types, the listener's address type will be looked up in a registry to find a
listener factory. The constructed listener will produce transports for new
connections satisfying a common interface.

### Client and Server Transports

Transport and stream APIs should expose gRPC semantics as defined in the
(pending) [Call Semantics
Specification](https://github.com/grpc/grpc/pull/15460) and hide HTTP/2
semantics and grpc's wire protocol semantics.

Stream APIs should support object-based messages to facilitate an in-process
transport based on copying message objects directly instead of serializing and
deserializing for transmission.  Note that some features, e.g. [client
retries](https://github.com/grpc/proposal/blob/master/A6-client-retries.md) may
require byte-based message support due to message caching. Such features may be
unsupported for transports that cannot also support byte-based messages.

The API for creating transports should encapsulate the logic for connecting to
addresses (e.g. net.Dial) as an implementation detail of the transport.
Similarly, credentials handshaking should happen within the transport.

### Temporary environment variable protection

Since all transport selection logic takes place in code and is not configured by
I/O, no environment variable is needed.

## Rationale

Alternatives considered:

* Hardcode support for all non-standard (non-HTTP/2) transports, like the
  in-memory transport.  Example `NewInMemoryChannel()`.

  This does not allow as much flexibility and requires recompilation to change
  the way channels are created.

* Require byte-based transports.

  This does not allow for a more efficient in-memory transport that is capable
  of copying proto messages rather than serializing and deserializing.

* Decide the transport directly when the channel is created, rather than
  allowing the resolver and balancer to influence it.

  This would have a simpler API, however, it would require each channel to use
  only a single transport. The proposed design enables a channel with different
  types of connections.

## Implementation

As this is a high-level, cross-language design, implementation design and
details will be left up to the gRPC library authors to handle independently.
