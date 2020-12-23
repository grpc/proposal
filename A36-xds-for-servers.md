A36: xDS-Enabled Servers
------------------------
* Author(s): [Eric Anderson](https://github.com/ejona86), [Doug
  Fawley](https://github.com/dfawley), [Mark Roth](https://github.com/markdroth)
* Approver: markdroth
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2020-11-19
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Bring xDS-based configuration to Servers. xDS has many features and this gRFC
will not provide support for any of them in particular. However, it will provide
the central plumbing and APIs to add support for specific features and have them
work without additional user code changes.

There's multiple pieces involved:

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
API. This will allow it to see the listening ports, start event, stop event, and
also allow it access to the server to change behavior (e.g., via an
interceptor). Users will use the XdsServer API to construct the server instead
of the existing API.

Credential configuration will be managed separately by providing
XdsServerCredentials, similar to client-side. The user must pass the
XdsServerCredentials to the XdsServer API to enable control-plane managed TLS
certificates. While the user-facing API is discussed here, the precise
implementation details are covered in gRFC A29. The user is free to pass other
ServerCredentials types to XdsServer and they will be honored. Note that RBAC is
authz, not authn, so it would be enabled by using the XdsServer, even without
XdsServerCredentials.

When starting the XdsServer, it will start communication with xDS to receive
initial configuration. `bind()` and `listen()` may happen immediately, but
`accept()`ing on the port must be delayed until the initial configuration is
received. `listen()` should be delayed if possible, but commonly needs to happen
immediately after `bind()` for implementation-specific reasons. If initial
configuration fails, the server will continue delaying and server startup will
stall until it succeeds. Each language will provide an XdsServer-specific API to
inform the application of initial communication failures. If the user does not
configure the API, gRPC should log the errors at a default-visible log level. If
the xds bootstrap is missing or invalid, implementations would ideally fail
server startup, but a permanent hang is also acceptable.

The XdsServer does not have to wait until server credentials (e.g., TLS certs)
are available before accepting connections; since XdsServerCredentials might not
be used, the server is free to lazily load credentials. However, the XdsServer
should keep the credentials cache fresh and up-to-date after that initial
lazy-loading, as it is clear at that point that XdsServerCredentials are being
used.

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
is not present in the bootstrap then server creation or start will fail.
The XdsServer will pass the listening address (commonly using a wildcard IP
address, like `::` or `0.0.0.0`) to a new XdsClient API to start a watch for a
`envoy.config.listener.v3.Listener` resource. XdsClient will perform the `%s`
replacement if the token is present and watch the corresponding listener
resource. No special character handling of the template or its replacement is
performed. For example, with an address of `[::]:80` and a template of
`grpc/server?xds.resource.listening_address=%s`, the resource name would be
`grpc/server?xds.resource.listening_address=[::]:80`.

The xDS-returned Listener must have an [`address`][Listener.address] that
matches the listening address provided. The Listener's `address` would be a TCP
`SocketAddress` with matching `address` and `port_value`. The XdsClient must
NACK the resource if the address does not match. The xDS client must also NACK
the resource if `Listener.listener_filters` is non-empty.

Although `FilterChain.filters`s will not be observed initially, the
`FilterChain` contains data that may be used like TLS configuration in
`transport_socket`. When looking for a FilterChain, the standard matching logic
must be used. Each `filter_chain_match` of the repeated
[`filter_chains`][Listener.filter_chains] should be checked for the specific
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

XdsClients may be shared without impacting this design. If shared, any mentions
of "creating" or "shutting down" a XdsClient would simply mean "acquire a
reference" and "release a reference" on the shared instance, or similar
behavior. Such sharing does not avoid the need of shutting down XdsClients when
no longer in use; they are a resource and must not be leaked.

This overall "wrapping" server API has many language-specific ramifications. We
show each individual language's approach.

### C++

C will need to expose an API for C++ to utilize. We expect there will be an xDS
filter that will be injected into the server. But we do not focus on that here
and leave that as an implementation detail. Since C is monolithic and we are
fine with xDS using internal APIs, there is quite a bit more flexibility in
design options available for C than Java and Go.

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

C will expose an API for wrapped languages to utilize. We do not focus on that
here and leave that as an implementation detail.

Each wrapped language will need a custom "xds server" creation API. This will
vary per-language. We use Python here just as an example of how it may look.

Python would add a `grpc.xds_server(...)` that mirrors `grpc.server(...)`. Since
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
installed, the builder may only be used once (`build()` should throw if called a
second time).

The xDS-aware server’s `start()` will create the `XdsClient` with the passed
port and wait for initial configuration before delegating to the real `start()`.
If the xDS bootstrap is missing it will throw an IOException within `start()`.
The `XdsClient` will be shut down on `shutdownNow()`/termination. When new or
updated configuration is received, it will create a new intercepter and update
the interceptor reference.

XdsServerBuilder will have an `xdsInitListener(XdsInitListener)`.
`XdsInitListener` will have the one method `xdsLoadFailure(IOException)` that is
called for failures loading the initial configuration within `start()`, since
`start()` itself does not fail in that case. A default implementation will log
the exception. If a user sets the listener, it will replace the default logging
implementation.

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

Create an `xds.GRPCServer` struct that would internally contain a `grpc.Server`.
It would inject its own `StreamServerInterceptor` and `UnaryServerInterceptor`s
into the `grpc.Server`. The interceptors would be controlled by an
`atomic.Value` or mutex-protected field that would update with the
configuration. When `Serve(net.Listener)` is called it creates an XdsClient and
observes the `Addr()` to create a watch.  Before `Serve(net.Listener)` returns,
it will shut down the XdsClient. Note that configuration is nominally
per-address, but there is only one set of interceptors, so the interceptors will
need to look up the per-address configuration each RPC.

TODO: It looks like there is a lifetime issue for the XdsClient, as RPCs can
continue after the listener is closed and Serve() returns.

Because service registration is done on a method in the generated code, and
`grpc.Server` is a struct, the underlying `grpc.Server` will need to be exposed.


```
// instead of
s := grpc.NewServer()
pb.RegisterGreeterServer(s, &server{})
// it'd be
s := xds.NewGRPCServer()
pb.RegisterGreeterServer(s.Server(), &server{})
```


To allow passing configuration, the `xds.Server` will be responsible for
creating the actual `XdsTransportCredentials`. There will be a façade
`XdsTransportCredentials` for the user to provide as a `ServerOption`.
`xds.Server` will look through the `ServerOptions`, do a type assertion on the
`TransportCredential`, and if it is the façade `XdsTransportCredentials` it will
replace the credentials with the real `XdsTransportCredential`. The real
`XdsTransportCredential` will need to use the local IP to find the port-specific
configuration.


## Rationale

XdsServer server startup is delayed waiting on initial configuration from xDS to
avoid applications getting hung in the rare event of an issue during startup. If
the start API was one-shot, then users would need to loop themselves and many
might introduce bugs in the rarely-run code path or simply not handle the case
at all.

Ideally `listen()` would be delayed until just before `accept()` to avoid
hanging new client connections. However, the kernel does not fully initialize
the port details until `listen()` and some gRPC implementations have behavior
and APIs that makes this infeasible. This may cause serious problems for
specific clients using pick-first but is essentially unavoidable today. The
impact is hoped to be low as servers will generally receive their initial
configuration quickly (on the order of a second) and most clients will likely be
using a variant of round-robin. We'd also encourge the xDS control plane to
avoid handing out addresses of not-yet-started servers. There are a few possible
remedies, for example using [Happy Eyeballs][RFC8305] for pick-first or having
the server `accept()` and immediately `close()` connections, but any such
mitigations would be future work.

[RFC8305]: https://tools.ietf.org/html/rfc8305

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

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

Go needs some final decisions made. C needs to be fleshed out.
