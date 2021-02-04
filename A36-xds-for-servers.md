A36: xDS-Enabled Servers
------------------------
* Author(s): [Eric Anderson](https://github.com/ejona86), [Doug
  Fawley](https://github.com/dfawley), [Mark Roth](https://github.com/markdroth)
* Approver: markdroth
* Status: In Review {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2020-11-19
* Discussion at: https://groups.google.com/g/grpc-io/c/CDjGypQi1J0

## Abstract

Bring xDS-based configuration to Servers. xDS has many features and this gRFC
will not provide support for any of them in particular. However, it will provide
the central plumbing and APIs to add support for specific features and have them
work without additional user code changes.

There are multiple pieces involved:

*  The API surface exposed to users to enable the new system.
*  The gRPC-internal plumbing to allow injecting behavior into servers, and
   changing that behavior on-the-fly.
*  The specific xDS protocol behavior gRPC will use to retrieve configuration.

## Background

Since [gRFC A27: xDS-Based Global Load Balancing][A27], Channels have had
growing support for understanding xDS configuration and configuring themselves.
However, Servers are also important to connect to xDS in order to power TLS
communication, authorization, fault injection, rate limiting, and other
miscellaneous features.

On client-side, users are able to opt-in to the behavior via the `xds:` scheme
which enables the xDS Name Resolver. Servers in gRPC do not use Name Resolution
and certainly don't have things like Load Balancing APIs which have been
leveraged heavily on the client-side design. While server-side will need to use
a different approach, users should have a similar experience: they make a
trivial code change to opt-in to xDS, and from that point on newly-added
xDS-powered gRPC features can be used without code changes.

### Related Proposals:

 * [A27: xDS-Based Global Load Balancing][A27]
 * [A29: xDS-Based Security for gRPC Clients and Servers][A29]

[A27]: A27-xds-global-load-balancing.md
[A29]: A29-xds-tls-security.md

## Proposal

Each language will create an "XdsServer" API that will wrap the normal "Server"
API. This will allow the implementation to see the listening ports, start event,
stop event, and also allow it access to the server to change behavior (e.g., via
an interceptor). Users will use the XdsServer API to construct the server
instead of the existing API. XdsServer will not support ephemeral ports as part
of this gRFC but may be enhanced in the future.

Credential configuration will be managed separately by providing
XdsServerCredentials, similar to client-side. The user must pass the
XdsServerCredentials to the XdsServer API to enable control-plane managed TLS
certificates. While the user-facing API is discussed here, the precise
implementation details are covered in gRFC A29. The user is free to pass other
ServerCredentials types to XdsServer and they will be honored. Note that RBAC is
authz, not authn, so it would be enabled by using the XdsServer, even without
XdsServerCredentials.

### Serving and Not Serving

To serve RPCs, the XdsServer must have its xDS configuration, provided via a
Listener resource and potentialy other related resources. When its xDS
configuration is unavailable the server must be in a "not serving" mode. This is
ideally implemented by not `listen()`ing on the port. If that is impractical an
implementation may be `listen()`ing on the port, but it must also `accept()` and
immediately `close()` connections, making sure to not send any data to the
client (e.g., no TLS ServerHello nor HTTP/2 SETTINGS). With either behavior,
client connection attempts will quickly fail and clients would not perform
further attempts without a backoff. Load balancing policies like `pick_first`
would naturally attempt connections to any remaining addresses to quickly find
an operational backend. However, the `accept()`+`listen()` approach will be
improperly detected as server liveness for TCP heath checking.

If the xDS bootstrap is missing or invalid, implementations would ideally fail
XdsServer startup, but it is also acceptable to consider it a lack of xDS
configuration and enter a permanent "not serving" mode.

If the server is unable to open serving ports (e.g., because the port is already
in use by another process), XdsServer startup may fail or it may enter "not
serving" mode. If entering "not serving" mode, opening the port must be retried
automatically (e.g., retry every minute).

Communication failures do not impact the XdsServer's xDS configuration; the
XdsServer should continue using the most recent configuration until connectivity
is restored. However, XdsServer must accept configuration changes provided by
the xDS server, including resource deletions. If that causes the XdsServer to
lack essential configuration (e.g., the Listener was deleted) the server would
need to enter "not serving" mode. "Not serving" requires existing connections to
be closed, but already-started RPCs should not fail. The XdsServer is permitted
to use the previously-existing configuration to service RPCs during a two-phase
GOAWAY to avoid any RPC failures, or it may use a one-phase GOAWAY which will
fail racing RPCs but in a way that the client may transparently retry. The
XdsServer is free to use a different "not serving" strategy post-startup than
for the initial startup.

The XdsServer does not have to wait until server credentials (e.g., TLS certs)
are available before accepting connections; since XdsServerCredentials might not
be used, the server is free to lazily load credentials. However, the XdsServer
should keep the credentials cache fresh and up-to-date after that initial
lazy-loading, as it is clear at that point that XdsServerCredentials are being
used.

The XdsServer API will allow applications to register a "serving state" callback
to be invoked when the server begins serving and when the server encounters
errors that force it to be "not serving". If "not serving", the callback must be
provided error information, for debugging use by developers. The error
information should generally be language-idiomatic, but determining the cause of
the error does not need to be machine-friendly. If the application does not
register the callback, XdsServer should log any errors and each serving
resumption after an error, all at a default-visible log level.

XdsServer's start must not fail due to transient xDS issues, like missing xDS
configuration from the xDS server. If XdsServer's start blocks waiting for xDS
configuration an application can use the serving state callback to be notified
of issues preventing startup progress.

### xDS Protocol

The `GRPC_XDS_BOOTSTRAP` file will be enhanced to have a new field:
```
{
  // A template for the name of the Listener resource to subscribe to for a gRPC
  // server. If the token `%s` is present in the string, it will be replaced
  // with the server's listening "IP:port" (e.g., "0.0.0.0:8080", "[::]:8080").
  "server_listener_resource_name_template": "example/resource/%s",
  // ...
}
```

XdsServer will use the normal `XdsClient` to communicate with the xDS server.
There is no default value for `server_listener_resource_name_template` so if it
is not present in the bootstrap then server creation or start will fail or the
XdsServer will become "not serving". XdsServer will perform the `%s` replacement
in the template (if the token is present) to produce a listener name. The
XdsServer will start an XdsClient watch on the listener name for a
`envoy.config.listener.v3.Listener` resource. No special character handling of
the template or its replacement is performed. For example, with an address of
`[::]:80` and a template of `grpc/server?xds.resource.listening_address=%s`, the
resource name would be `grpc/server?xds.resource.listening_address=[::]:80`.

To be useful, the xDS-returned Listener must have an
[`address`][Listener.address] that matches the listening address provided. The
Listener's `address` would be a TCP `SocketAddress` with matching `address` and
`port_value`. The XdsServer must be "not serving" if the address does not match.

The xDS client must NACK the Listener resource if `Listener.listener_filters` is
non-empty.

### FilterChainMatch

Although `FilterChain.filters`s will not be used as part of this gRFC, the
`FilterChain` contains data that may be used like TLS configuration in
`transport_socket`. When looking for a FilterChain, the standard matching logic
must be used. The most specific `filter_chain_match` of the repeated
[`filter_chains`][Listener.filter_chains] should be found for the specific
connection and if none match, the `default_filter_chain` must be used. The
following is a snippet of [all current fields of
FilterChainMatch][FilterChainMatch], and if they will be handled specially:

[Listener.address]: https://github.com/envoyproxy/envoy/blob/11dee4e9c37c244f9de4bc339fdc8695e5de2c5d/api/envoy/config/listener/v3/listener.proto#L108
[Listener.filter_chains]: https://github.com/envoyproxy/envoy/blob/11dee4e9c37c244f9de4bc339fdc8695e5de2c5d/api/envoy/config/listener/v3/listener.proto#L117
[FilterChainMatch]: https://github.com/envoyproxy/envoy/blob/928a62b7a12c4d87ce215a7c4ebd376f69c2e080/api/envoy/config/listener/v3/listener_components.proto#L85

```
message FilterChainMatch {
  enum ConnectionSourceType {
    ANY = 0;
    SAME_IP_OR_LOOPBACK = 1;
    EXTERNAL = 2;
  }
  google.protobuf.UInt32Value destination_port = 8; // Always fail match
  repeated core.v3.CidrRange prefix_ranges = 3;
  ConnectionSourceType source_type = 12;
  repeated core.v3.CidrRange source_prefix_ranges = 6;
  repeated uint32 source_ports = 7;
  repeated string server_names = 11; // Always fail match
  string transport_protocol = 9; // Only matches "raw_buffer"
  repeated string application_protocols = 10; // Always fail match
}
```

All fields are "supported," however, we know that some features separate from
the match are unsupported. Fields depending on missing features are guaranteed a
result that can be hard-coded. This applies to `destination_port` which relies
on `use_original_dst`. It also applies to `server_names`, `transport_protocol`,
`application_protocols` which depend on `Listener.listener_filters`.

### Language-specifics

The overall "wrapping" server API has many language-specific ramifications. We
show each individual language's approach. For C++ and wrapped languages we just
show a rough sketch of how they would be done.

XdsClients may be shared without impacting this design. If shared, any mentions
of "creating" or "shutting down" an XdsClient would simply mean "acquire a
reference" and "release a reference" on the shared instance, or similar
behavior. Such sharing does not avoid the need of shutting down XdsClients when
no longer in use; they are a resource and must not be leaked.

### C++

This section is a sketch to convey the "feel" of the API. But details may vary.

C will need to expose an API for C++ to utilize. We expect there will be an xDS
filter that will be injected into the server. But we do not focus on that here
and leave that as an implementation detail to be part of a future gRFC. Since C
is monolithic and we are fine with xDS using internal APIs, there is quite a bit
more flexibility in design options available for C than Java and Go.

Create an `XdsServerBuilder` that mirrors the `ServerBuilder` API. It may be
possible to implement it via delegating to a `ServerBuilder` instance, but that
is an implementation detail.

The `XdsServerBuilder` will need to use a C core API (e.g., a channel arg to
pass a plugin instance) to enable xDS support in the built `Server`. The
`XdsServerBuilder` can indirectly create the `XdsClient` and plumb it to filters
or other code that may need it. Notably, it will _not_ be passed to the
`ServerCredential`; the `ServerCredential` will be passed the configuration to
use for each connection, like `ChannelCredential`, so won’t need to use the
`XdsClient` for watches on the ADS stream. Since many `Server` methods are not
`virtual`, the `Server` will _not_ be wrapped to add xDS functionality. This
means the C server will need to be responsible for shutting down the
`XdsClient`.

Since `ServerBuilder` has few virtual methods, the `XdsServerBuilder` will not
be directly interchangeable. We believe users will be minimally impacted by
needing to refer to the precise type, especially with the availability of
templates. If this becomes a problem more methods could become `virtual`.


### Wrapped Languages

This section is a sketch to convey the "feel" of the API. But details may vary.

C will expose an API for wrapped languages to utilize. We do not focus on that
here and leave that as an implementation detail to be part of a future gRFC.

Each wrapped language will need a custom "xds server" creation API. This will
vary per-language. We use Python here just as an example of how it may look.

Python could add a `grpc.xds_server(...)` that mirrors `grpc.server(...)`. Since
ports are added after the server is created, the returned `grpc.Server` object
may be an xds-aware `Server` to coordinate with the C API when `server.start()`
is called. However, its implementation should be trivial as it could mostly
delegate to a normal `server` instance.

It would be possible to manage the lifetime of the XdsClient from the `server`,
based on `server.stop()`, but given C++ lacks this avenue the C API probably
will not need this notification.


### Java

Create an `XdsServerBuilder` that extends `ServerBuilder` and delegates to a
"real" builder. The server returned from `builder.build()` would be an xDS-aware
server delegating to a "real" server instance. The `XdsServerBuilder` would
install an `AtomicReference<ServerInterceptor>`-backed interceptor in the
built server and pass the listening port and interceptor reference to the
xDS-aware server when constructed. Since interceptors cannot be removed once
installed, the builder may only be used once; `build()` should throw if called a
second time. To allow the `XdsServerBuilder` to create addition servers if
necessary, mutation of the builder after `build()` should also throw.

The xDS-aware server’s `start()` will create the `XdsClient` with the passed
port and wait for initial configuration before delegating to the real `start()`.
If the xDS bootstrap is missing it will throw an IOException within `start()`.
The `XdsClient` will be shut down when the server is terminated (generally
noticed via `shutdownNow()`/`awaitTermination()`). When new or updated
configuration is received, it will create a new intercepter and update the
interceptor reference.

XdsServerBuilder will have an
`xdsServingStatusListener(XdsServingStatusListener)` method.
`XdsServingStatusListener` will be defined as:

```java
package io.grpc.xds;

public interface XdsServingStatusListener {
  void onServing();
  void onNotServing(Throwable t);
}
```

It is an interface instead of an abstract class as additional methods are not
expected to be added before Java 8 language features are permitted in the code
base. If this proves incorrect, an additional interface can be added.

If not specified, a default implementation of `XdsServingStatusListener` will be
used. It will log the exception and log calls to `onServing()` following a call
to `onNotServing()` at WARNING level.

In order to allow transport-specific configuration, the `XdsServerBuilder` will
have a `@ExperimentalApi ServerBuilder transportBuilder()` method whose return
value can be cast to `NettyServerBuilder` for experimental API configuration.

`NettyServerBuilder` will add an internal method `eagAttributes(Attributes)`. It
will plumb those attributes to `GrpcHttp2ConnectionHandler.getEagAttributes()`
as implemented by `NettyServerHandler`, which currently just returns
`Attributes.EMPTY`. `XdsServerBuilder` will then specify `eagAttributes` to
inject credential information for the `XdsServerCredential`, similar to
client-side, although with the added step of needing to process the
FilterChainMatch for the specific connection.


### Go

Create an `xds.GRPCServer` struct that would internally contain an unexported
`grpc.Server`. It would inject its own `StreamServerInterceptor` and
`UnaryServerInterceptor`s into the `grpc.Server`. The interceptors would be
controlled by an `atomic.Value` or mutex-protected field that would update with
the configuration. 

`GRPCServer.Serve()` takes a `net.Listener` instead of a `host:port` string to
listen on, so as to be consistent with the `Serve()` method on the
`grpc.Server`. The implementation expects the `Addr()` method on the passed in
`net.Listener` to return an address of type `net.TCPAddr`. It then creates an
XdsClient and observes the `Addr()` to create a watch.  Before
`Serve(net.Listener)` returns, it will cancel the watch registered on the
XdsClient. As part of `Stop()/GracefulStop()`, the XdsClient is shut down. Note
that configuration is nominally per-address, but there is only one set of
interceptors, so the interceptors will need to look up the per-address
configuration for each RPC.

Service registration is done on a method in the generated code which
accepts a `grpc.ServiceRegistrar` interface. Both `grpc.Server` and 
`xds.GRPCServer` implement this interface and therefore the latter can be passed to
service registration methods just like the former.


```
// instead of
s := grpc.NewServer()
pb.RegisterGreeterServer(s, &server{})
// it'd be
s := xds.NewGRPCServer()
pb.RegisterGreeterServer(s, &server{})
```


Package `credentials/xds` exports a `NewServerCredentials()` function which
returns a transport credentials implementation which uses security configuration
received from the xDS server. The user is expected to provide this credentials
as part of the `grpc.ServerOption`s passed to `xds.NewGRPCServer()`, if they are
interested in sourcing their security configuration from the xDS server. It is
the responsibility of the `xds.GRPCServer` implementation to forward the most
recently received security configuration to the credentials implementation.

## Rationale

XdsServer server startup does not fail due to configuration from xDS server to
avoid applications getting hung in the rare event of an issue during startup. If
the start API was one-shot, then users would need to loop themselves and many
might introduce bugs in the rarely-run code path or simply not handle the case
at all.

We chose the XdsServer approach over an "xDS Interceptor." In such a design, the
user would just construct an XdsInterceptor and add it to their server.
Unfortunately the interceptor would not be informed of the server lifecycle and
so could not manage the connection to the xDS control plane. It also is not
informed of listening ports until RPCs actually arrive on those ports and is not
able to learn when a port is no longer in use (as is possible in grpc-go). It
would also be difficult to share the XdsClient configuration between the
interceptor and credentials. It simply cannot be used in its current state.

We chose the XdsServer approach over creating a new server-side plugin API. In
such a design we could create a new plugin API where the user could construct
the Xds module and add it to the server. The events needing to be exposed are:
server start, server stop, listening socket addition, listening socket removal.
Those events are basically exactly what are on the server API today, making this
very similar functionally to wrapping, but in an awkward and repetitive way. It
would also need some method of plumbing the XdsClient for the Xds
ServerCredential to use, which is unclear how that would be accomplished.

## Implementation

All prototyping work occurred in Java so it is furthest along, with only smaller
changes necessary to converge with the design presented here. Go is expected to
be complete very soon after Java. C++/wrapped languages are expected to lag
waiting on the C core implementation.

* C++/wrapped languages. The surface APIs should be relatively easy compared to
  the C core changes. The C core changes are being investigated by @markdroth
  and the plan is to flesh them out later when we have a better idea of the
  appropriate structure. So the gRFC just tries to show that the API design
  would work for C++/wrapped languages but there is currently no associated
  implementation work.

* Java. Implementation work primarily by @sanjaypujare, along with gRFC A29.
  Added classes are in the `grpc-xds` artifact and `io.grpc.xds` package.

* Go. Implementation work by @easwars