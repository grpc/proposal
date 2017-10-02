Title
----
* Author(s): mehrdada
* Approver: nathanielmanistaatgoogle
* Status: Draft
* Implemented in: Python
* Last updated: October 2, 2017
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal suggests an interceptor facility for gRPC Python.

## Background

gRPC Python does not currently have built-in support for intercepting calls
that is present in some other gRPC languages, like Java, Go, and Ruby.

This proposal introduces a client and server interceptor facility for
gRPC Python.  The client interceptors are used to create a decorated
channel that intercepts client gRPC calls and server interceptors act
as decorators over handlers.


### Related Proposals

* [Ruby Client and Server Interceptors](https://github.com/grpc/proposal/pull/34)
* [C# Client and Server Interceptors](https://github.com/grpc/proposal/pull/38)

## Proposal

This proposal introduces the machinery required for implementing
interceptors and hooking them up to the gRPC Python framework.

### Client Interceptors

On the client side, interception occurs by creating an intercepted channel
based on an underlying `grpc.Channel` object and a set of interceptors.
Stubs can then be created on top of this new decorated channel and be used
in the application as usual. RPCs dispatched from such stubs then go
through the interceptor chain for processing before they hit the
underlying gRPC channel.  This is done through the following API:

```python
intercept_channel(   # returns intercepted `grpc.Channel`
    channel          # underlying `grpc.Channel`
    *interceptors)   # interceptors to add
```

The interceptors passed to `intercept_channel` should derive from one or
more of the following abstract classes depending on the types of RPCs they
are interested in intercepting:

```python
grpc.UnaryUnaryClientInterceptor           # intercepts unary-unary calls
grpc.UnaryStreamClientInterceptor          # intercepts unary-stream calls
grpc.StreamUnaryClientInterceptor          # intercepts stream-unary calls
grpc.StreamStreamClientInterceptor         # intercepts stream-stream calls
```

The above abstract classes each require implementation of one of the
following methods, respectively:

```python
# grpc.UnaryUnaryClientInterceptor:
intercept_unary_unary_call(self, invoker, method, request, **kwargs)
intercept_unary_unary_future(self, invoker, method, request, **kwargs)

# grpc.UnaryStreamClientInterceptor:
intercept_unary_stream_call(self, invoker, method, request, **kwargs)

# grpc.StreamUnaryClientInterceptor:
intercept_stream_unary_call(self, invoker, method, request_iterator, **kwargs)
intercept_stream_unary_future(self, invoker, method, request_iterator, **kwargs)

# grpc.StreamStreamClientInterceptor:
intercept_stream_stream_call(self, invoker, method, request_iterator, **kwargs)
```

As you can see, the interception functions operating on stream
requests (`intercept_stream_*_call`), take an iterator of request
values instead of a singular request value as an argument.

An alternate option would have been making an interceptor a callable
object.  However, this design has the advantage that a single class
can implement all of the interceptor types at once, and in fact, due
to deliberate name differences in the methods, a single class can
be an invocation-side and a service-side interceptor at once.

Due to the nature of the interceptors operating on RPCs
returning streams (`intercept_*_stream_call`),
they are inherently asynchronous and do not have a blocking ("synchronous")
mode.  However, interceptors operating on the RPCs returning a unary
(`intercept_*_unary_call`), intercept blocking calls, and have a
corresponding asynchronous version (`intercept_unary_unary_future`,
`intercept_stream_unary_future`).  A unary-unary and a stream-unary
client interceptor needs to implement both of these functions.
The `intercept_*_unary_call` version returns the RPC response and
the `grpc.Call` value associated with the RPC, whereas the
`intercept_*_unary_future` returns a future object which upon
completion, will carry the reponse value of the RPC.

All of the methods are passed a callable value `invoker`, whose
return type matches the return type of that is expected of the
function it is passed to. `invoker` is the continuation that
progresses the RPC forward by calling the next interceptor
in the chain, or the core RPC invoker, in case of the last
interceptor.  It is the responsibility of the interceptor
to call `invoker` should it chooses to continue with the RPC,
but is not required to do so.  The ability to invoke the
`invoker` not exactly once is the reason this design is
advantageous over the system automatically invoking
the continuation upon returning from interceptor.

`invoker` takes the arguments
`method, request/request_iterator, **kwargs`, where
`method` is the method name of the RPC, `request` or
`request_iterator` are represent the request value(s) of
the RPC, and `**kwargs` represents the additional arguments
like `metadata`, `timeout`, and `credentials`, passed to
the RPC.  All of these arguments, including the method called,
can be changed by the interceptor and it is free to change
any or all of those arguments when passing them to `invoker`,
but it would be natural to pass the additional `**kwargs`
that the interceptor does not particularly understand
or care about unmodified.


### Server Interceptors

On the server side, interceptors are registered on the server
by creating an intercepted gRPC server via the following API:

```python
intercept_server(       # returns intercepted grpc.Server
    server,             # underlying grpc.Server instance
    *interceptors)      # interceptors to add
```

The returned server object, upon getting requests to add
handler objects (`grpc.GenericRpcHandler`), will attach
the interceptor chain registered on it to the handler,
and registers that combined handler on the underlying
`grpc.Server`, thereby passing control to the interceptors
instead of directly letting the handler service the call.

It is therefore possible to intercept calls to some handlers
registered on a server object and leave others plain.
To accomplish that, simply create an intercepted server
and register the service handlers that you want to intercept
on top of the intercepted server, but use the underlying
server directly to register the handlers that are not supposed
to be intercepted.

Similar to the client interceptors, on the server side, there
are four distinct abstract classes, and implementing at least
one of them is essential for every gRPC server interceptor:

 ```python
grpc.UnaryUnaryServerInterceptor           # intercepts unary-unary calls
grpc.UnaryStreamServerInterceptor          # intercepts unary-stream calls
grpc.StreamUnaryServerInterceptor          # intercepts stream-unary calls
grpc.StreamStreamServerInterceptor         # intercepts stream-stream calls
```

which require one of the following functions, respectively:

```python
intercept_unary_unary_handler(self, handler, method, request, servicer_context)
intercept_unary_stream_handler(self, handler, method, request, servicer_context)
intercept_stream_unary_handler(self, handler, method, request_iterator, servicer_context)
intercept_stream_stream_handler(self, handler, method, request_iterator, servicer_context)
```

Similar to the client-side interceptors, RPCs with streaming request type,
get an iterator instead of a singular request value, and the RPCs with
streaming response types are expected to return a response iterator value.
On the server side, `handler` is analogous to the `invoker` on the client
side and continues with the RPC servicing. The return value of `handler`
is of the same type as the expected return value of the particular
`intercept_*_handler` function it is passed to, and `handler` takes
`(request/request_iterator, servicer_context)` as arguments.

Unlike the client side, `method` is read-only on the server-side and
it is not possible for the interceptor to change that.

## Rationale

In addition to some design decisions which were were justified in-line
with the proposal description above, the proposal has opted to provide
hooks of maximal granularity in the form of distinct functions for
each RPC type and blocking/asynchony characteristic of the invocations.
While many interceptors may choose to implement common logic in many
of those cases, that unification is best done at a higher level point,
*e.g.* a utility class that adapts the common logic to all RPC types,
to give more flexibility to the user when they need to write specialized
code when necessary.

## Implementation

Implemented in [gRPC Python Client and Server Interceptors pull request][impl]

[impl]: https://github.com/grpc/grpc/pull/12778