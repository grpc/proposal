Node: Make `Server#start` a no-op
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: In Review
* Implemented in: Node.js
* Last updated: 2023-10-02
* Discussion at: https://groups.google.com/g/grpc-io/c/zcGg6ipiZMo

## Abstract

Remove the functionality of the `start` method in the `Server` class, and deprecate that method.

## Background

Currently, a `Server` object effectively has a two-phase startup sequence: first, the user calls `bindAsync` to bind a port, then they call `start` to start serving. Specifically, `bindAsync` puts the server into a state where it allows clients to establish HTTP/2 sessions on the specified port, but then immediately closes those sessions. `start` causes the server to stop closing those sessions, and to instead serve requests on them.

This behavior is problematic because allowing clients to establish HTTP/2 sessions signals to the client that the server is available, and clients will continue trying to connect to the same server instead of trying other servers. In addition, clients do not back off when reconnecting after a connection that was successfully established is closed, so this behavior will cause clients continuously reconnect with no backoff.

## Proposal

The behavior of `bindAsync` will be modified, so that the server will begin handling requests immediately after the port is bound. `start` will become a no-op, except that it will throw an error in the same situations when it currently does, for compatibility with code that expects those errors. `start` will also be deprecated, which means that it will output a standard Node deprecation message once per process.

An incidental consequence of this change is that the rule that `bindAsync` cannot be called after `start` will be removed.

## Rationale

The state we currently provide between `bindAsync` and `start` has little practical use in a running server, but can cause significant behavior and performance degredation if reached by accident (e.g. by omitting the `start` call). So it would be best to avoid those mistakes entirely by not providing that option. This change is potentially breaking if anyone is deliberately using that state, but I think that is unlikely. Anyone who is currently calling `start` in the callback for `bindAsync` should see minimal behavioral difference with this change.

The Node API for this is very simple, making it difficult to find a useful alternative behavior for `start`. The `Http2Server` class has a `listen` method, which binds and listens on a port, and automatically accepts incoming connections (and performs the TLS handshake, if applicable) and exchanges `SETTINGS` frames to establish the HTTP/2 session. Once that is complete, the gRPC gets access to the session object in a `session` event. At that point, the session has already been successfully established from the client's point of view, and the gRPC code can't undo that. The `Http2Server` class also has a `connection` event, which provides access to a `Socket` object representing the TCP socket. Closing the connection at that point results in the client seeing `ECONNRESET` instead of a `GOAWAY`, but the client still considers the connection to have been established, and does not behave differently as a result.


## Implementation

I (murgatroid99) will implement this in the Node gRPC library.
