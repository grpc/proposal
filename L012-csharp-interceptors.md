Title
----
* Author(s): mehrdada
* Approver: jtattermusch
* Status: Approved
* Implemented in: C#
* Last updated: June 7, 2018
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
functionality is effectively non-existent.


### Related Proposals:

* [Ruby Client and Server
  Interceptors](https://github.com/grpc/proposal/pull/34) proposal
* [Node Client Interceptors](https://github.com/grpc/proposal/pull/14) proposal
* [Python Client and Server
  Interceptors](https://github.com/grpc/proposal/pull/39) proposal

## Proposal

This proposal consists of two pieces of infrastructure for server and client
interceptors, both of which are placed in the new `Grpc.Core.Interceptors`
namespace in `Grpc.Core` assembly.

While client and server interceptors use different hooks to intercept on RPC
invocations, they share the abstract base class `Interceptor` without any
abstract methods.  An alternative was initially considered where two separate
base classes supported client and server interceptors. The rationale for
settling on a single base class was to enable a single class to be able to get
registered both as a server interceptor and a client interceptor, making
libraries providing generic interceptors nicer to use, and the cost of having
several additional virtual methods in the same class was minimal should one
want to implement only one side of the interceptor.  Another alternative
considered was using an `interface` instead of an abstract base class, but that
was ruled out since interfaces are less amenable to non-breaking changes than
abstract classes across future versions (adding a method to an interface would
be a breaking change for existing users, whereas adding an additional method to
an abstract base class should be fine as long as it is not marked `abstract`).

### Abstract Base Class

All interceptors must derive, directly or indirectly, from the abstract base
class `Grpc.Core.Interceptors.Interceptor` and can choose to override zero or
more of its methods.

### Client Interceptors

The `Interceptor` class defines the following virtual methods to hook into RPC
invocations of various types on the client side:

```csharp
TResponse BlockingUnaryCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, BlockingUnaryCallContinuation<TRequest, TResponse> continuation);
AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, AsyncUnaryCallContinuation<TRequest, TResponse> continuation);
AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, AsyncServerStreamingCallContinuation<TRequest, TResponse> continuation);
AsyncClientStreamingCall<TRequest, TResponse> AsyncClientStreamingCall<TRequest, TResponse>(ClientInterceptorContext<TRequest, TResponse> context, AsyncClientStreamingCallContinuation<TRequest, TResponse> continuation);
AsyncDuplexStreamingCall<TRequest, TResponse> AsyncDuplexStreamingCall<TRequest, TResponse>(ClientInterceptorContext<TRequest, TResponse> context, AsyncDuplexStreamingCallContinuation<TRequest, TResponse> continuation);
```

Four of these methods correspond to the asynchronous invocations of the four
different possible RPC types supported by gRPC: unary, server-streaming,
client-streaming, and duplex-streaming; the fifth, `BlockingUnaryCall` is
provided for performance reasons for non-asynchronous unary-unary invocations.
The API is designed to directly correspond to the methods supported by
`CallInvoker`.  An interceptor implementation interested in intercepting each
RPC type needs to override and customize the implementation of each of the
respective methods. Once again, blocking and asynchronous code paths for unary
invocations are separate and both need to be implemented and overriding one
does not automatically apply to the other.

Of the aformentioned methods, the ones that intercept request-unary RPCs take
an instance of the unary request object.  All methods take a context class of
type `Grpc.Core.Interceptors.ClientInterceptorContext` and a `continuation`
delegate to invoke to continue with the RPC chain.

#### The `ClientInterceptorContext` class

A new class, `Grpc.Core.Interceptors.ClientInterceptorContext` is introduced
to encapsulate the details of the invocation, namely the following properties,
but is flexible to support the addition of new properties in the future:

- `Method` of type `Method<TRequest, TResponse>` representing the current
  method to be invoked.
- `Host`, a string representing the host name of the current invocation.
- `Options`, the `CallOptions` for the current invocation.

A `context` instance is passed to the interceptor to describe the invocation.
The interceptor is free to propagate the same object or create its own context
object and pass it to the next continuation to control how the invocation is
going to proceed.

An alternative was considered to pass such properties in-line as arguments
to the interceptor and have the interceptor invoke the continuation listing
them as arguments, but in addition to usability hurdles due to verboseness,
having a class makes adding properties more seamless and future-proof than
adding a parameter to method signatures, thus that alternative was
eliminated.

#### The `continuation` delegate

The invocation-side interceptors get control over an RPC invocation and can
read and optionally modify aspects of the invocation they are interested in.
Such properties are passed via the request value argument for the unary RPCs
and the context object specified above.  In order to proceed with the
execution of the RPC, the interceptor can choose to call the `continuation`
and pass it a new request instance (for request-unary RPCs only) and a context
instance.  The interceptor is allowed to invoke `continuation` zero (useful
for a caching interceptor, for example) or more times (e.g. useful for a retry
interceptor) and return a value as it sees fit.

In the general case, the interceptors are not obligated to invoke `continuation`
just once.  They might choose to not continue with the call chain and terminate
it immediately by returning their own desired return value, or potentially call
`continuation` more than once, to simulate retries, transactions, or other
similar potential use cases.  In fact, the ability to take an explicit
`continuation` argument is the primary rationale for needing a special-purpose
interceptor machinery on the client side, since `CallInvoker` is capable of
doing a lot for us.  That said, chaining `CallInvoker`s require each one to
explicitly hold a reference to the next `CallInvoker` in the chain.  This
design, however, abstracts away the next function from the object and passes it
as a continuation to each interceptor hook.  An internal `CallInvoker`
implementation chains interceptors together and ultimately with the underlying
`CallInvoker` and `Channel` objects.  All of the functionality described so far
could have been implemented as an external library, and does not change the
non-intercepted code path at all, therefore should not have any negative
performance impact when no interceptor is used.

#### Return values

The return value of each method is of the same type as the corresponding
method in the `CallInvoker` object for that RPC type and is documented in
that class.  For asynchronous and streaming invocations, intercepting the
entire call would entail returning a custom instance of the `Async***Call`
value that wraps the value returned from `continuation`.

#### Registration

Client interceptors are registered on a `Channel` or `CallInvoker` via extension
methods named `Intercept` defined in `ChannelExtensions` and
`CallInvokerExtensions`.  While they fundamentally operate on an underlying
`CallInvoker` and not a `Channel` directly, the ones that take a `Channel` as an
argument serve as syntactic sugar for more ergonomy in application code and they
simply create an underlying `DefaultCallInvoker` and route calls through it.
Since `Intercept` is the only way to register interceptors on a `CallInvoker`,
the `InterceptingCallInvoker` that chains interceptor and exposes a
`CallInvoker` can remain a `private` class.

An alternative design registering interceptors directly on a `Channel` returning
an "intercepted `Channel`" object was considered but ultimately dismissed,
despite having an advantage of 100% compatibility with anywhere a `Channel`
would have been used, since it added complexity to the `Channel` object and
potentially slowing down the fast code path where no interceptor is registered.
When invoking RPCs in user code, `CallInvoker` is sufficiently close to a
`Channel` for constructing client objects and using local variable type
inference (`var`), the actual user code would look identical in most cases:

```csharp
var interceptedChannel = channel.Intercept(interceptorObject);
// interceptedChannel type is really `CallInvoker`, but since
// both `CallInvoker` and `Channel` can be passed to the generated
// client code, the code looks similar.
var greeterClient = new GreeterClient(interceptedChannel);
```

A higher-level, more user-friendly, canonical base class for common interceptor
functionality can be provided by deriving from `Interceptor` and adding unified
hooks (e.g. `BeginCall`, `EndCall`) that can be overridden once but operate on
all types of RPC invocations.  Additionally, a pipeline-builder style of
registration can be added in the future.  Since this gRFC does not preclude
addition of those, and in the interest of not adding APIs that would be set
in stone, the scope of this gRFC is limited to the core interceptor hooks and
external libraries can provide such functionality for now.

#### Order of execution

Registering an interceptor on a `Channel`-like object is to be thought of as
treating the underlying object as a black box and intercept methods before
they are handed over to the underlying object.  Thus, intercepting an
intercepted channel will result in the last interceptor being run first:

```csharp
var chan = newChannel();
var interceptedOnce = chan.Intercept(interceptor1);
var interceptedTwice = interceptedOnce.Intercept(interceptor2);
// interceptor2 will take control first, and calling continuation
// from interceptor2 will invoke interceptor1. continuation passed
// to interceptor1 will invoke the call invoker which invokes
// the channel.
```

However, a helper API is provided to register multiple interceptors
at once on a `Channel` or `CallInvoker`, in which case, the order
of execution is in the order listed.

```csharp
var interceptedChannel = chan.Intercept(interceptor1, interceptor2);
// interceptor1 will take control before interceptor2.
```

#### Simple client interceptor example

```csharp
// Invokes the specified callback before each RPC gets invoked.
class CallbackInterceptor : Interceptor
{
    readonly Action callback;
    public CallbackInterceptor(Action callback)
    {
        this.callback = GrpcPreconditions.CheckNotNull(callback, nameof(callback));
    }
    public override TResponse BlockingUnaryCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, BlockingUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        callback();
        return continuation(request, context);
    }
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        callback();
        return continuation(request, context);
    }
    public override AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(TRequest request, ClientInterceptorContext<TRequest, TResponse> context, AsyncServerStreamingCallContinuation<TRequest, TResponse> continuation)
    {
        callback();
        return continuation(request, context);
    }
    public override AsyncClientStreamingCall<TRequest, TResponse> AsyncClientStreamingCall<TRequest, TResponse>(ClientInterceptorContext<TRequest, TResponse> context, AsyncClientStreamingCallContinuation<TRequest, TResponse> continuation)
    {
        callback();
        return continuation(context);
    }
    public override AsyncDuplexStreamingCall<TRequest, TResponse> AsyncDuplexStreamingCall<TRequest, TResponse>(ClientInterceptorContext<TRequest, TResponse> context, AsyncDuplexStreamingCallContinuation<TRequest, TResponse> continuation)
    {
        callback();
        return continuation(context);
    }
}
```


### Server Interceptors

Server-side interceptors derive from the `Interceptor` class and optionally
override the four relevant methods to intercepting different kinds of
incoming RPCs.  All of these methods are expected to return a `Task` object
and thus can be asynchronous in nature.

```csharp
Task<TResponse> UnaryServerHandler<TRequest, TResponse>(TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation);
Task<TResponse> ClientStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, ServerCallContext context, ClientStreamingServerMethod<TRequest, TResponse> continuation);
Task ServerStreamingServerHandler<TRequest, TResponse>(TRequest request, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, ServerStreamingServerMethod<TRequest, TResponse> continuation);
Task DuplexStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, DuplexStreamingServerMethod<TRequest, TResponse> continuation);
```

Of these four, the two that return unary values are expected to return a
a `TResponse` value asynchronously (via `Task<TResponse>`) whereas the
response-streaming RPCs are expected to write the return values to the
`responseStream` stream.  Additionally, for unary requests, the request
value is passed as an argument to the intercepting method directly,
whereas for client-streaming RPCs, they should be read from the
`requestStream` given to the interceptor.

#### The RPC context

The interceptor is passed a `context` object representing the
context of the current RPC.  This is the same object passed to
RPC handlers and the interceptor is in a position to treat
itself as if it were the actual RPC handler.

#### The `continuation` delegate

A `continuation` delegate is passed to the server-side interceptor
method that matches the signature of the actual RPC handler (depending
on the RPC kind), and its purpose is to carry-on with RPC processing.

The `continuation` takes the same parameters as the interceptor method,
with the exception of the `continuation` itself, and returns a value of
the same type as the intercepting method.

#### Intercepting requests and responses

The intercepting method can choose to inspect and modify
the request value as it passes it to the `continuation` (for the
request-unary RPCs), optionally inspect and modify the response
value returned from `continuation` invocation (for response-unary RPCs).
Additionally, it can do sophisticated interception over the lifecycle
of streaming RPCs by wrapping the `requestStream` (for client-streaming
RPCs) and/or `responseStream` (for server-streaming RPCs).

#### Registration

Server interceptors are registered on `ServerServiceDefinition` objects
via the `Intercept` extension method provided by `ServerSeviceDefinition`
method.

```csharp
Server server = new Server
{
  Services = { Greeter.BindService(new GreeterImpl()).Intercept(new LogInterceptor()) },
  Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
};
server.Start();
```

#### Order of execution

Server interceptors operate identically to the client interceptors in
terms of order of execution.  Please consult the section above for details.


#### Simple server-side interceptor example

```csharp
// The interceptor invokes the registered callback on every RPC
// and passes the context value of the current call to it
class ServerCallContextInterceptor : Interceptor
{
    readonly Action<ServerCallContext> interceptor;

    public ServerCallContextInterceptor(Action<ServerCallContext> interceptor)
    {
        GrpcPreconditions.CheckNotNull(interceptor, nameof(interceptor));
        this.interceptor = interceptor;
    }

    public override Task<TResponse> UnaryServerHandler<TRequest, TResponse>(TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation)
    {
        interceptor(context);
        return continuation(request, context);
    }

    public override Task<TResponse> ClientStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, ServerCallContext context, ClientStreamingServerMethod<TRequest, TResponse> continuation)
    {
        interceptor(context);
        return continuation(requestStream, context);
    }

    public override Task ServerStreamingServerHandler<TRequest, TResponse>(TRequest request, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        interceptor(context);
        return continuation(request, responseStream, context);
    }

    public override Task DuplexStreamingServerHandler<TRequest, TResponse>(IAsyncStreamReader<TRequest> requestStream, IServerStreamWriter<TResponse> responseStream, ServerCallContext context, DuplexStreamingServerMethod<TRequest, TResponse> continuation)
    {
        interceptor(context);
        return continuation(requestStream, responseStream, context);
    }
}
```

## Rationale

A primary design goal is not breaking the existing API at all.  In addition,
flexibility, decoupling and non-disruption to existing code, and keeping the
"fast path", where no interceptor is registered, fast. Some design trade-offs
to accomplish these goals were discussed in-line with the decision.

## Implementation

[C# Interceptor Support Pull Request](https://github.com/grpc/grpc/pull/12613)
