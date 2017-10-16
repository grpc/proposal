Title
----
* Author(s): mehrdada
* Approver: jtattermusch
* Status: In Review
* Implemented in: C#
* Last updated: October 16, 2017
* Discussion at: https://groups.google.com/d/topic/grpc-io/GKcnFC3oxhk/discussion

## Abstract

This proposal introduces client and server interceptor APIs in gRPC C#.

The API provides the facility to register certain functions to run when an RPC
is invoked on the client or the server side.  These functions can be chained and
composed flexibly as users wish.

## Background

gRPC C# does not have an interceptor API.  Functionality can be partially
simulated on the client side by extending the `CallInvoker` class. However,
this has some limitations in composition and the ability to decouple
`CallInvoker` from each other.  On the server side, similar or replacement
functionality was basically non-existent.


### Related Proposals:

* [Ruby Client and Server
  Interceptors](https://github.com/grpc/proposal/pull/34) proposal
* [Node Client Interceptors](https://github.com/grpc/proposal/pull/14) proposal
* [Python Client and Server
  Interceptors](https://github.com/grpc/proposal/pull/39) proposal

## Proposal

This proposal consists of two independent pieces of infrastructure for server
and client interceptors, both of which are rooted in the new
`Grpc.Core.Interceptors` namespace in `Grpc.Core` assembly.

### Client Interceptors

Client interceptors derive from the new abstract base class
`Grpc.Core.Interceptors.ClientInterceptor`.  This abstract class defines five
different hooks for various RPC types and invocations, namely
`BlockingUnaryCall`, `AsyncUnaryCall`, `AsyncClientStreamingCall`,
`AsyncServerStreamingCall`, and `AsyncDuplexStreamingCall`.  The signature of
these methods correspond to each of their counterparts in `CallInvoker`, with
the notable distinction that each take an additional argument `next`, which is
a delegate that invokes the next step in the interceptor chain or the actual
underlying `CallInvoker` handler for the final interceptor in the chain:

```csharp
TResponse BlockingUnaryCall<TRequest, TResponse>(Method<TRequest, TResponse> method, string host, CallOptions options, TRequest request, Func<Method<TRequest, TResponse>, string, CallOptions, TRequest, TResponse> next);
AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(Method<TRequest, TResponse> method, string host, CallOptions options, TRequest request, Func<Method<TRequest, TResponse>, string, CallOptions, TRequest, AsyncUnaryCall<TResponse>> next);
AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(Method<TRequest, TResponse> method, string host, CallOptions options, TRequest request, Func<Method<TRequest, TResponse>, string, CallOptions, TRequest, AsyncServerStreamingCall<TResponse>> next);
AsyncClientStreamingCall<TRequest, TResponse> AsyncClientStreamingCall<TRequest, TResponse>(Method<TRequest, TResponse> method, string host, CallOptions options, Func<Method<TRequest, TResponse>, string, CallOptions, AsyncClientStreamingCall<TRequest, TResponse>> next);
AsyncDuplexStreamingCall<TRequest, TResponse> AsyncDuplexStreamingCall<TRequest, TResponse>(Method<TRequest, TResponse> method, string host, CallOptions options, Func<Method<TRequest, TResponse>, string, CallOptions, AsyncDuplexStreamingCall<TRequest, TResponse>> next);
```

In the general case, the interceptors are not obligated to invoke `next` once.
They might choose to not continue with the call chain and terminate it
immediately by returning their own desired return value, or potentially call
`next` more than once, to simulate retries, transactions, or other similar
potential use cases.  In fact, the ability to take an explicit `next` argument
is the primary rationale for needing a special-purpose interceptor machinery on
the client side, since `CallInvoker` is capable of doing a lot for us.
However, chaining `CallInvoker`s require each one to explicitly hold a
reference to the next `CallInvoker` in the chain.  The `ClientInterceptor`
design, however, abstracts away the next function from the object and passes it
as a continuation to each interceptor hook.  An internal `CallInvoker`
implementation chains interceptors together and ultimately with the underlying
`CallInvoker` and `Channel` objects.  All of the functionality described so far
could have been implemented as an external library, and does not change the
non-intercepted code path at all.  In addition to hook functions to be
overridden by the client interceptor implementations, the class provides a few
`static` methods that lets the interceptor register additional hooks to execute
code when certain events happen in asynchronous and streaming calls.  These
helper static functions are the only reason this facility could not have been
implemented by an external library: they require access to the internal
constructors of `AsyncUnaryCall<TResponse>`,
`AsyncServerStreamingCall<TResponse>`, and so on:

```csharp
protected static AsyncUnaryCall<TResponse> Intercept<TRequest, TResponse>(AsyncUnaryCall<TResponse> call,
    Func<Task<TResponse>, TResponse> response = null,
    Func<Task<Metadata>, Metadata> responseHeaders = null,
    Func<Func<Status>, Func<Status>> getStatus = null,
    Func<Func<Metadata>, Func<Metadata>> getTrailers = null,
    Func<Action, Action> dispose = null);

protected static AsyncServerStreamingCall<TResponse> Intercept<TRequest, TResponse>(AsyncServerStreamingCall<TResponse> call,
    Func<IAsyncStreamReader<TResponse>, IAsyncStreamReader<TResponse>> responseStream = null,
    Func<Task<Metadata>, Metadata> responseHeaders = null,
    Func<Func<Status>, Func<Status>> getStatus = null,
    Func<Func<Metadata>, Func<Metadata>> getTrailers = null,
    Func<Action, Action> dispose = null);

protected static AsyncClientStreamingCall<TRequest, TResponse> Intercept<TRequest, TResponse>(AsyncClientStreamingCall<TRequest, TResponse> call,
    Func<IClientStreamWriter<TRequest>, IClientStreamWriter<TRequest>> requestStream = null,
    Func<Task<TResponse>, TResponse> response = null,
    Func<Task<Metadata>, Metadata> responseHeaders = null,
    Func<Func<Status>, Func<Status>> getStatus = null,
    Func<Func<Metadata>, Func<Metadata>> getTrailers = null,
    Func<Action, Action> dispose = null);

protected static AsyncDuplexStreamingCall<TRequest, TResponse> Intercept<TRequest, TResponse>(AsyncDuplexStreamingCall<TRequest, TResponse> call,
    Func<IClientStreamWriter<TRequest>, IClientStreamWriter<TRequest>> requestStream = null,
    Func<IAsyncStreamReader<TResponse>, IAsyncStreamReader<TResponse>> responseStream = null,
    Func<Task<Metadata>, Metadata> responseHeaders = null,
    Func<Func<Status>, Func<Status>> getStatus = null,
    Func<Func<Metadata>, Func<Metadata>> getTrailers = null,
    Func<Action, Action> dispose = null);
```

Client interceptors are registered on a `Channel` or `CallInvoker` via
extension methods named `Intercept` defined in `ChannelExtensions` and
`CallInvokerExtensions`.  While they fundamentally operate on an underlying
`CallInvoker` and not a `Channel` directly, the ones that take a `Channel` as
an argument serve as a syntactic sugar for ease of construction in user code:
they simply create an underlying `DefaultCallInvoker` and route calls through
it.  Since `Intercept` is the only way to register interceptors on a
`CallInvoker`, the `InterceptingCallInvoker` that chains interceptor and
exposes a `CallInvoker` can remain an internal class.

An alternative design registering interceptors directly on a `Channel`
returning an "intercepted `Channel`" object was considered but ultimately
dismissed, despite having an advantage of 100% compatibility with anywhere a
`Channel` would have been used, since it added complexity to the `Channel`
object and potentially slowing down the fast code path where no interceptor is
registered.  When invoking RPCs in user code, `CallInvoker` is sufficiently
close to a `Channel` for constructing client objects and using local variable
type inference (`var`), the actual user code would look identical in most
cases:

```csharp
var interceptedChannel = channel.Intercept(interceptorObject);
// interceptedChannel type is really `CallInvoker`, but since
// both `CallInvoker` and `Channel` can be passed to the generated
// client code, the code looks similar.
var greeterClient = new GreeterClient(interceptedChannel);
```

A higher-level, more user-friendly, canonical base class for common interceptor
functionality can be provided by deriving from `ClientInterceptor` and adding
unified hooks (e.g. `BeginCall`, `EndCall`) that can be overridden once but
operate on all types of RPC invocations.

In order to carry satellite data across interceptors and `CallInvoker`s, this
proposal suggests `CallOptions` to be amended with an `Items` dictionary and a
`WithItems` helper method to register a custom dictionary which would hold
shared state across the interceptors and `CallInvoker`s.  This is the only other
piece of client interceptors that requires being implemented within the
`Grpc.Core` since it needs adding a member to `CallOptions` structure.


### Server Interceptors

Server-side interceptors derive from the new `ServerInterceptor` class
implemented in `Grpc.Core.Interceptors` namespace.  Server interceptors are
registered on individual service definitions as opposed to an entire server,
though it might be sensible to provide a syntactic sugar for adding an
interceptor chain to all services served by a `Server` instance.

In particular, an instance of a class derived from `ServerInterceptor` is
registered via an extension method for class `ServerServiceDefinition`.  This
method is defined as an extension method to decouple the server interceptor
machinery from the core gRPC infrastructure as much as possible.  This
extension method wraps all of the existing handlers in the service definition
with ones intercepted by the passed interceptor object.

```csharp
Server server = new Server
{
	Services = { Greeter.BindService(new GreeterImpl()).Intercept(new LogInterceptor()) },
	Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
};
server.Start();
```

Since support for this handler substitution was needed from the
`ServerServiceDefinition` class itself, a general internal method
`SubstituteHandlers` is added to actually apply the mapping given the wrapping
logic as a delegate and return a new service definition object with the new
handlers.  Beyond this, and slight modification to the definition of server
call handlers, no special support from the core gRPC machinery was needed, and
the fast path without any registered interceptor remains intact.

Similar to client interceptors, server interceptors can override four functions
`UnaryServerHandler`, `ClientStreamingServerHandler`,
`ServerStreamingServerHandler`, and `DuplexStreamingServerHandler`.  Likewise,
they all take a `next` argument that invokes the next interceptor in the chain
or the actual RPC handler at the end of the chain:

```csharp
async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> next);
async Task<TResponse> ClientStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, ServerCallContext context, ClientStreamingServerMethod<TRequest, TResponse> next);
async Task ServerStreamingServerHandler<TRequest, TResponse>(TRequest request, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, ServerStreamingServerMethod<TRequest, TResponse> next);
async Task DuplexStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, DuplexStreamingServerMethod<TRequest, TResponse> next);
```

In order to carry satellite data between interceptors and the handler, this
proposal suggests adding an `Items` dictionary to `ServerCallContext` to hold
arbitrary state.

## Rationale

The design goals where flexibility, decoupling and non-disruption to existing
code, and keeping the fast path fast and the design trade-offs made intend to
accomplish these goals.  Some of these decisions were discussed above.


## Implementation

[C# Interceptor Support Pull Request](https://github.com/grpc/grpc/pull/12613)
