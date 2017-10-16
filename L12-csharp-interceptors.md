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
simulated on the client side by extending the `CallInvoker` class. However, this
has some limitations in composition and the ability to decouple `CallInvoker`
from each other.  On the server side, similar or replacement functionality was
basically non-existent.


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
abstract methods.  The rationale for this is it enables a single class to act
both as a server interceptor and a client interceptor.  An alternative
considered were using an `interface` instead of an abstract base class, but were
ruled out due to potential future versioning issues (adding a method to an
interface would be a breaking API change, whereas adding an additional method to
an abstract base should be fine as long as it is not marked `abstract`). Another
alternative that was ruled out was to separate it out to `ClientInterceptor` and
`ServerInterceptor` classes.  This may have made each class look slightly more
lightweight,  but realistically, it would be just as easy to implement just the
client methods and not the server methods on an interceptor.

### Client Interceptors

Client interceptors derive from the new abstract base class
`Grpc.Core.Interceptors.Interceptor`.  This abstract class defines five
different virtual methods to hook into various RPC invocation types, namely
`BlockingUnaryCall`, `AsyncUnaryCall`, `AsyncClientStreamingCall`,
`AsyncServerStreamingCall`, and `AsyncDuplexStreamingCall`.

The request-unary interceptor hooks take a request value as their first
argument, followed by the common arguments for all of the methods:

1. A `context` argument of a new type `ClientInterceptorContext<TRequest,
   TResponse>` that encapsulates the context of the call.  An older alternative
   design used a similar signature to the corresponding method in the
   `CallInvoker` class, but that would limit the ability to add new contextual
   information to pass along without changing the signature and thus breaking
   API compatibility.

2. A `continuation` argument, which is a delegate that invokes the next step in
   the interceptor chain or the actual underlying `CallInvoker` handler for the
   final interceptor in the chain and returns a value of the same type as the
   interceptor method.  For unary requests, it takes the `request` value as its
   first argument, followed by the `context` to proceed the invocation with. The
   interceptor method is allowed to pass in a new or modified `context` along
   and that is what the RPC continues with.  For client-streaming RPCs, only the
   `context` argument should be passed along to the `continuation`.

```csharp
TResponse BlockingUnaryCall<TRequest, TResponse>(
    TRequest request,
    ClientInterceptorContext<TRequest, TResponse> context,
    BlockingUnaryCallContinuation<TRequest, TResponse> continuation)

AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
    TRequest request,
    ClientInterceptorContext<TRequest, TResponse> context,
    AsyncUnaryCallContinuation<TRequest, TResponse> continuation)

AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(
    TRequest request,
    ClientInterceptorContext<TRequest, TResponse> context,
    AsyncServerStreamingCallContinuation<TRequest, TResponse> continuation)

AsyncClientStreamingCall<TRequest, TResponse> AsyncClientStreamingCall<TRequest, TResponse>(
    ClientInterceptorContext<TRequest, TResponse> context,
    AsyncClientStreamingCallContinuation<TRequest, TResponse> continuation)
```
AsyncDuplexStreamingCall<TRequest, TResponse> AsyncDuplexStreamingCall<TRequest, TResponse>(
    ClientInterceptorContext<TRequest, TResponse> context,
    AsyncDuplexStreamingCallContinuation<TRequest, TResponse> continuation)
```

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
all types of RPC invocations.


### Server Interceptors

Server-side interceptors derive from the new `Interceptor` class implemented in
`Grpc.Core.Interceptors` namespace.  Server interceptors are registered on
individual service definitions as opposed to an entire server, though it might
be sensible to provide a mechanism for adding an interceptor chain to all
services served by a `Server` instance in the future.

In particular, an instance of a class derived from `Interceptor` is registered
on a `ServerServiceDefinition` instance via its new `Intercept` method, which
returns a new `ServerServiceDefinition` whose handlers are intercepted through
the given interceptor.  An alternate design was considered in which `Intercept`
was an extension method on `ServerServiceDefinition` and reached out to an
internal `SubstituteHandlers` method on the object for minimal disruption to
`ServerServiceDefinition`.  However, the benefits of decoupling it was unclear,
since the class defining that extension method would have needed to access the
internals of `ServerServiceDefinition` nevertheless.

```csharp
Server server = new Server
{
  Services = { Greeter.BindService(new GreeterImpl()).Intercept(new LogInterceptor()) },
  Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
};
server.Start();
```

The handler substitution process operates on each of the four RPC handler types,
and registers the interceptor on them.  The RPC handler dispatch code is changed
so that if at least one interceptor is registered on the handler, the
interceptor chain is invoked and is given the call context.  This singular
check, i.e. whether or not at least one interceptor is registered for a handler,
is the only additional work needed be done on the fast path, where no
interceptor is registered, so the performance impact should be minimal and
contained to one additional `interceptor == null` check for each RPC invocation.
If an interceptor chain exists, it is then given control and is allowed to
substitute the call handler with an arbitrary continuation, which is invoked
instead of the handler and can intercept the call during its full duration.  The
first interceptor invocation is also allowed to return the original handler
intact, indicating it not being interested to do any additional processing or
observation on the call, or throw an exception and terminate the RPC
immediately.  Throwing an exception from an interceptor is semantically
equivalent to throwing an exception from a server handler.

The method signatures for the server side interceptor hooks are as follows:

```csharp
// THandler is one of:
// Grpc.Core.UnaryServerMethod<TRequest, TResponse>,
// Grpc.Core.ClientStreamingServerMethod<TRequest, TResponse>,
// Grpc.Core.ServerStreamingServerMethod<TRequest, TResponse>, or
// Grpc.Core.DuplexStreamingServerMethod<TRequest, TResponse>,
// depending on the RPC type.
delegate Task<THandler> ServerHandlerInterceptor<THandler>(ServerCallContext context, THandler handler);

ServerHandlerInterceptor<UnaryServerMethod<TRequest, TResponse>> GetUnaryServerHandlerInterceptor<TRequest, TResponse>()
ServerHandlerInterceptor<ServerStreamingServerMethod<TRequest, TResponse>> GetServerStreamingServerHandlerInterceptor<TRequest, TResponse>()
ServerHandlerInterceptor<ClientStreamingServerMethod<TRequest, TResponse>> GetClientStreamingServerHandlerInterceptor<TRequest, TResponse>()
ServerHandlerInterceptor<DuplexStreamingServerMethod<TRequest, TResponse>> GetDuplexStreamingServerHandlerInterceptor<TRequest, TResponse>()
```

By default, the implementation is a no-op:

```csharp
return (context, handler) => Task.FromResult(handler);
```

The design in which all four server interceptor methods were combined into one
was considered, but dismissed in favor of this one because eventually, the
interceptor hook needed to return an object of the same type as the handler
passed to it, and it would have needed to reflect over it to reconstruct the
type.  Instead, under this design, such unification is still an option, simply
by deriving a class that implements all four functions and calls a shared method
to minimize the code written, should that style fits that particular application
better.


## Rationale

A primary design goal is not breaking the existing API at all.  In addition,
flexibility, decoupling and non-disruption to existing code, and keeping the
"fast path", where no interceptor is registered, fast. Some design trade-offs to
accomplish these goals were discussed in-line with the decision.


## Implementation

[C# Interceptor Support Pull Request](https://github.com/grpc/grpc/pull/12613)
