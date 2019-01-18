gRPC Python Interceptors
----
* Author(s): lidizheng
* Approver: ericgribkoff
* Status: In Review
* Implemented in: Python
* Last updated: January 17, 2019
* Discussion at: https://groups.google.com/d/msg/grpc-io/7qIzyCPWbUA/1NKGXAQHCgAJ

## Abstract

This proposal is a retroactive design document that explains the design and implementation details of the interceptor functionality for gRPC Python, which was merged in 2017. This gRFC serves as an effort to stabilize the interceptor APIs.


## Background

### The Need for Interceptors

Interceptors are middleware which is capable of manipulating or reacting to the transporting requests/responses throughout the life cycle of an RPC. It dramatically simplifies the implementation of tracing (See [grpc/grpc#8767](https://github.com/grpc/grpc/issues/8767)) and empowers gRPC developers to add general channel-level business logic.

Interceptors have existed within gRPC for a while now, and its semantics were implemented in bottom-up implementations like Java, Go and wrapper implementations like Node.js, Ruby.

### Related Proposals

* [Node.js Client Interceptor](https://github.com/grpc/proposal/blob/master/L5-node-client-interceptors.md)
* [Ruby Client and Server Interceptors](https://github.com/grpc/proposal/pull/34)
* [C# Client and Server Interceptors](https://github.com/grpc/proposal/pull/38)


## Proposal

### Interface: Client Call Details

To intercept an RPC, we first require an interface by which a developer may inspect and modify options pertaining to a specific call. Hence we introduce the new interface `grpc.ClientCallDetails`, which contains five properties:

```Python
class ClientCallDetails(six.with_metaclass(abc.ABCMeta)):
    """Describes an RPC to be invoked.

    Attributes:
      method: The method name of the RPC.
      timeout: An optional duration of time in seconds to allow for the RPC.
      metadata: Optional metadata to be transmitted to the service-side of the RPC.
      credentials: An optional CallCredentials for the RPC.
      wait_for_ready: This is an EXPERIMENTAL argument. An optional flag to enable wait for ready mechanism.
    """
```

### Concept: Continuation

The `continuation` function is a function that proceeds with the invocation by executing the next interceptor in the chain or invoking the actual RPC on the underlying Channel. It is the interceptor’s responsibility to call the `continuation` unless it decides to terminate processing of the RPC. The interceptor can use `response_future = continuation(client_call_details, request)` to continue with the RPC. The `continuation` returns an object that is both a Call for the RPC and a Future. In the event of RPC completion, the return Call-Future’s result value will be the response message of the RPC. Should the event terminate with non-OK status, the returned Call-Future’s exception value will be an RpcError.

### Client-Side Interceptors

There are four interceptor interfaces mapping to the four possible cardinalities of an RPC. These interfaces enables developers to implement an arbitrary number of interceptor type within one class with the Python inheritance semantic. Each interceptor interface will have a corresponding abstract class method that takes in `continuation`, `grpc.ClientCallDetails`, and the outgoing RPC requests. The return value of that class method will be both a `grpc.Call` and a `grpc.Future` or response iterator depends on the cardinality of RPC. The definition of interfaces are:

```Python
class UnaryUnaryClientInterceptor(six.with_metaclass(abc.ABCMeta)):
    @abc.abstractmethod
    def intercept_unary_unary(self, continuation, client_call_details,
                              request): pass

class UnaryStreamClientInterceptor(six.with_metaclass(abc.ABCMeta)):
    @abc.abstractmethod
    def intercept_unary_stream(self, continuation, client_call_details,
                               request): pass

class StreamUnaryClientInterceptor(six.with_metaclass(abc.ABCMeta)):
    @abc.abstractmethod
    def intercept_stream_unary(self, continuation, client_call_details,
                               request_iterator): pass

class StreamStreamClientInterceptor(six.with_metaclass(abc.ABCMeta)):
    @abc.abstractmethod
    def intercept_stream_stream(self, continuation, client_call_details,
                                request_iterator): pass
```

To apply the interception, we provide a function that wraps the original channel with interceptors.

```Python
def intercept_channel(channel, *interceptors):
    """Intercepts a channel through a set of interceptors.

    Args:
      channel: A Channel.
      interceptors: Zero or more objects of type
        UnaryUnaryClientInterceptor,
        UnaryStreamClientInterceptor,
        StreamUnaryClientInterceptor, or
        StreamStreamClientInterceptor.
        Interceptors are given control in the order they are listed.

    Returns:
      A Channel that intercepts each invocation via the provided interceptors.
```

### Client-Side Implementation

The actual RPC of a gRPC client is conducted through `*MultiCallable` objects (see [API](https://grpc.io/grpc/python/grpc.html#multi-callable-interfaces)), which handles the encoding/decoding and coupling with the remote servicer method (through the method string). In the most common use case, the gRPC Stub is a class with initialized `*MultiCallable` objects as member variables. So, to intercept the invocation to the original `*MultiCallable` object without changing application code, the interceptor will create a `grpc.Channel` instance that wraps the creation of `*MultiCallable` object. The wrapper layer will intercept the calling arguments, pack them into a `grpc.ClientCallDetails` instance, then passes it to the interceptor.

This mechanism means that the same number as interceptors of extra layers of `grpc.Channel` will be created during the interception. The interceptors process each outgoing request/message in order. So, to quickly understand the idea, the execution flow will be:

```
Application Invoke an RPC ->
    Interceptor A Start ->
        Interceptor B Start ->
            Interceptor C Start ->
                Invoke Original '*MultiCallable' ->
                Return the Response from the Server ->
            Interceptor C Returns ->
        Interceptor B Returns ->
    Interceptor A Returns ->
Application Gets the Response
```

Also, this implementation empowers developers to invoke the `Continuation` zero or more times if necessary, which is useful in case of authentication or retry.

### Server-Side API

On the server side, there is also an RPC description class, which only contains two properties, `method` and `invocation_metadata`. Unlike the client side, the `method` variable is read-only.

```Python
class HandlerCallDetails(six.with_metaclass(abc.ABCMeta)):
    """Describes an RPC that has just arrived for service.

    Attributes:
      method: The method name of the RPC.
      invocation_metadata: The metadata sent by the client.
    """
```

The `Continuation` in the server side is thinner than the client side. It is a function that takes a `grpc.HandlerCallDetails` and proceeds to invoke the next interceptor in the chain, if any, or the RPC handler lookup logic, with the call details, passed as an argument and returns a `grpc.RpcMethodHandler` instance if the RPC is considered serviced, or None otherwise.

The server-side interceptor interface contains one abstract member function that takes in `Continuation`, and `grpc.HandlerCallDetails`, then returns a `grpc.RpcMethodHandler`.

```Python
class ServerInterceptor(six.with_metaclass(abc.ABCMeta)):
    @abc.abstractmethod
    def intercept_service(self, continuation, handler_call_details): pass
```

To apply interception, the [server construction function](https://grpc.io/grpc/python/grpc.html#grpc.server) takes in an `interceptors` parameter that accepts optional list of `grpc.ServerInterceptor` instance. The usage snippet will look like:

```Python
class ExampleInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        ...

server_interceptor_1 = ExampleInterceptor(...)
server = grpc.server(..., interceptors=[server_interceptor_1])
```

### Server-Side Implementation

In the implementation, there is a new concept named "Service Pipeline". It assembles the server side interceptors and is responsible for generating the final `grpc.RpcMethodHandler` to be invoked for incoming requests. The generation process will happen every time the server receives an RPC, the execution flow will look like:

```
Server Receives a Request ->
    Interceptor A Start ->
        Interceptor B Start ->
            Interceptor C Start ->
                The Original Handler
            Interceptor C Returns Updated Handler C ->
        Interceptor B Returns Updated Handler B ->
    Interceptor A Returns Updated Handler A ->

    Invoke the Updated Handler A with the Request ->
    Updated Handler A Returns Response ->
Server Replies
```

The most significant difference compared to the client side is that the interception happens ahead of the actual invocation of application behavior. So the interceptor function will be able to access neither the request nor servicer context.

## Rationale

### The Benefit of Introducing "Continuation"

Quote from mehrdada@ [design doc](https://github.com/grpc/proposal/blob/361846400834164d21c12ca6cf60e8cbf501d6cd/L13-Python-Interceptors.md):
> All of the methods are passed a callable value `Continuation`, whose
> return type matches the return type of that is expected of the
> function it is passed to. `Continuation` is the continuation that
> progresses the RPC forward by calling the next interceptor
> in the chain, or the core RPC Continuation, in case of the last
> interceptor.  It is the responsibility of the interceptor
> to call `Continuation` should it chooses to continue with the RPC,
> but is not required to do so.  The ability to invoke the
> `Continuation` not exactly once is the reason this design is
> advantageous over the system automatically invoking
> the continuation upon returning from interceptor.

### History of Its Implementation

The Python interceptors were requested by joeblubaugh@ from Twitter in [grpc/grpc#8767](https://github.com/grpc/grpc/issues/8767) in 2016 November. According to the discussion, the Python teams decided that this feature should be done by the community, since the `grpc.Channel` interface can be easily re-implemented. 

Fortunately, rnburn@ was willing to implement this feature to push the progress of [OpenTracing](https://opentracing.io/), and submitted a pull request [grpc/grpc#10256](https://github.com/grpc/grpc/pull/10256) in 2017 March. The pull request defines the majority of the semantics of today's gRPC Python interceptors implementation, which wraps over the existing abstraction of `grpc.Channel` and `grpc.Server`.

However, rnburn@ realized (see [comment](https://github.com/grpc/grpc/pull/10256#issuecomment-311495482)) that this approach prevents the capability for gRPC Python to provide same interceptor granularity as other implementation (Java, Node.js), and halt the work on this PR. To provide the most effective interceptor semantics, the implementation has to wrap over low-level C-Core API like `startBatch`, which sends batches of gRPC operations into C-Core. The potential amount of work is much more considerable than expected, it might require to refactor abstractions of gRPC or have hacky injections of logic.

To balance the cost and benefit, the Python team decided to carry on with the approach of wrapping over high-level abstraction. Then, merhdada@ wrote a gRFC ([grpc/proposal#39](https://github.com/grpc/proposal/pull/39)) that extends the functionality of interceptor along with two PR ([grpc/grpc#12778](https://github.com/grpc/grpc/pull/12778) and [grpc/grpc#13722](https://github.com/grpc/grpc/pull/13722)) based on rnburn@'s implementation. However, the PR itself conflicts with the gRFC, more specifically, the client-side interceptor API is changed, and the server-side interceptor semantics are not fully implemented.

### Client-Side Interceptor Semantic Comparison

Among the various implementations across different languages, there are two distinct interceptor designs.

1. Wraps over low-level abstraction like startBatch, Operations (Java, Node.js).
2. Wraps over high-level abstraction like Server, Channel (Python, Golang).

The interceptor API of the first approach enables gRPC developers to add handlers to the various event, take [Node.js Interceptor design](https://github.com/grpc/proposal/blob/master/L5-node-client-interceptors.md) as an example:

1. __start__: An interception method called before an outbound call has started.
2. __sendMessage__: An interception method called prior to every outbound message.
3. __halfClose__: An interception method called when the outbound stream closes (after the message is sent).
4. __cancel__: An interception method called when the stream is canceled from the client.
5. __onReceiveMetadata__: An inbound interception method triggered when metadata is received.
6. __onReceiveMessage__: An inbound interception method triggered when each message is received.
7. __onReceiveStatus__: An inbound interception method triggered when status is received.

The interceptor API of the second approach requires developers to wrap logic over the invocation of the RPC call, take [Python unary-unary Interceptor](https://grpc.io/grpc/python/grpc.html#client-side-interceptor) as an example:

```Python
class _ExampleClientInterceptor(grpc.UnaryUnaryClientInterceptor):

    def intercept_unary_unary(self, continuation, client_call_details, request):
        new_details, new_request = <Pre Process Logic>

        response = continuation(new_details, new_request)

        new_response = <Post Process Logic>
        return new_response  # It's both a grpc.Call and a grpc.Future.

```

The difference between mehrdada@'s gRFC and implementation is that: 1) the function signature changed; 2) the actual implementation doesn't support asynchronous interception; 3) the concept of `invoker` is replaced by `continuation`; 4) the actual implementation packs credentials/timeout/wait_for_ready/method/metadata into `grpc.ClientCallDetails`.

### Server-Side Interceptor Semantic Comparison

[Java server interceptor](https://grpc.io/grpc-java/javadoc/) allows developers to register event handlers:

1. **onCancel**: The call was cancelled and the server is encouraged to abort processing to save resources, since the client will not process any further messages.
2. **onComplete**: The call is considered complete and onCancel() is guaranteed not to be called.
3. **onHalfClose**: The client completed all message sending.
4. **onMessage**: A request message has been received.
5. **onReady**: This indicates that the call is now capable of sending additional messages without requiring excessive buffering internally.

Golang server interceptor and Python server interceptor **by design** will achieve the similar functionality of the client interceptor that the interception logic is handled inside one function. Here is the definition of Golang's unary server interceptor:

```Golang
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
// Takes in:
// 1. Context
// 2. Request
// 3. ServerInfo (Server itself & the method path)
// 4. handler (The actual handler of the request)
// Outputs:
// 1. Response
// 2. error
```

However, the actual Python implementation only allows developers to interact with the invocation metadata, or route the RPC to a newly defined method handler. It has no control over the invocation of actual user-defined request handler, hence the response message and the raised exception will not be propagated to the interceptor function. Here is an example of Python server interceptor:

```Python
class _ExampleServerInterceptor(grpc.ServerInterceptor):

    def intercept_service(self, continuation, handler_call_details):
        if ('custom-auth-header', 'secretkey') in handler_call_details.invocation_metadata:
            return continuation(handler_call_details)
        else:
            return grpc.unary_unary_rpc_method_handler(lambda request, context: context.abort(<code>, <details>))
```

## Implementation

[grpc/grpc#12778](https://github.com/grpc/grpc/pull/12778) and [grpc/grpc#13722](https://github.com/grpc/grpc/pull/13722)
