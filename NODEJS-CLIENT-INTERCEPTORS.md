NodeJS Client Interceptors
----
* Author(s): David Vroom-Duke (dduke@netflix.com), William Thurston (wthurston@netflix.com), David Liu (dliu@netflix.com), Howard Yuan (hyuan@netflix.com), Eran Landau (elandau@netflix.com)
* Approver: a11r
* Status: Draft
* Implemented in: Java, Go
* Last updated: 2017-02-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

NodeJS gRPC clients will present an interceptor interface. Consumers of these clients can provide an ordered list of
interceptors with methods to be executed on any outbound or inbound operation. The NodeJS client interceptor framework
API will provide similar functionality as existing gRPC interceptor implementations.

## Background

Some gRPC language implementations currently have client interceptor support and some do not. The NodeJS implementation
does not have client interceptor support, which makes the application of cross-cutting concerns difficult.

### Related Proposals:

None

## Proposal

### Interceptor API
Interceptors must implement `interceptCall`:

```javascript
/**
 * @param {Function} callConstructor A constructor for a grpc.Call proxy object
 * @param {object} options
 * @param {MethodDescriptor} options.method_descriptor A container of properties describing the call (see below)
 * @param {grpc.ChannelCredentials} [options.credentials] Credentials used to authenticate with the server
 * @param {number} [options.deadline] The deadline for the call
 * @param {string} [options.host] The server to call
 * @param {grpc.Call} [options.parent] The parent call
 * @param {number} [options.propagate_flags] Propagation flags for the underlying grpc.Call
 * @return {object} A grpc.Call proxy
 */
interceptCall(callConstructor, options)
```

The `interceptCall` method provides interceptors an opportunity to return a replacement for the interceptor's
`grpc.Call` proxy (which may include interceptor methods). An `InterceptorBuilder` will be provided to easily construct
an interceptor.

`interceptCall` must return an object with a `grpc.Call` proxy interface. The proxy interface is identical to the
underlying `grpc.Call` interface, except for the addition of a `context` argument to the `startBatch` method.
Returning `callConstructor(options)` will satisfy the contract (but provide no interceptor functionality).

Most interceptors will use the `ForwardingCall` constructor to create a call proxy with interceptor methods. Returning
`new ForwardingCall(callConstructor(options), outboundInterceptor)` will cause the call to trigger any interceptor
methods in the `outboundInterceptor` (described below).

A short-circuiting interceptor can return a custom `grpc.Call` proxy which skips any downstream interceptors. A simple
example is:
```javascript
{
    interceptCall: function(callConstructor, options) {
        var call = callConstructor(options);
        var _originalStartBatch = call.startBatch;
        call.startBatch = function(batch, context, cb) {
            var cachedResponse = _getCachedResponse(batch);
            if (cachedResponse) {
                cb(null, {
                    read: cachedResponse,
                    metadata: {},
                    status: {
                        code: grpc.status.OK,
                        metadata: {}
                    }
                });
            } else {
                _originalStartBatch(batch, context, cb);
            }
        };
        return call;
    }
}
```
The `grpc.Call` proxy interface is identical to the `grpc.Call` interface except for the `startBatch` signature. A
`context` argument has been added to carry values needed to serialize/deserialize messages.

```javascript
/**
* @param {object} client_batch
* @param {object} context
* @param {Function} callback
*/
startBatch(client_batch, context, callback)

```

The `options` argument to `interceptCall` includes all the options that go into the base `grpc.Call` constructor, plus
an additional `MethodDescriptor` object. The `MethodDescriptor` is a container for properties of the call which are 
used internally and may also be useful to consumers:

```javascript
/**
 * @param {string} name The name of the call, i.e. 'myCall'
 * @param {string} path The full path of the call, i.e. '/MyService/MyCall'
 * @param {MethodType} method_type One of four types: MethodType.UNARY, MethodType.CLIENT_STREAMING,
 * MethodType.SERVER_STREAMING, MethodType.BIDI_STREAMING
 * @param {Function} serialize The function used to serialize a message
 * @param {Function} deserialize The function used to deserialize a message
 * @constructor
 */
MethodDescriptor(name, path, method_type, serialize, deserialize)
```

An outbound interceptor is a POJO which implements zero or more of the following methods:

```javascript
/**
 * An interceptor method called before an outbound call has started.
 * @param {Metadata} metadata The call's outbound metadata (request headers).
 * @param {function} next A callback which continues the gRPC interceptor chain.
 * The first argument to next is required: a Metadata object.
 * The second argument to next is optional: an inbound interceptor (described below)
 */
start(metadata, next)

/**
 * An interceptor method called prior to every outbound message.
 * @param {object} A protobuf message
 * @param {function} next A callback which continues the gRPC interceptor chain, called with the message to send.
 */
sendMessage(message, next)

/**
 * An interceptor method called when the outbound stream closes (after the message is sent).
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
halfClose(next)

/**
 * An interceptor method called when the stream is canceled from the client.
 * @param {message} string|null A cancel message if provided
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
cancel(message, next)

```

An `OutboundInterceptorBuilder` will be provided to easily construct an outbound interceptor.

An inbound interceptor is an object passed to the `start` method's `next` callback with zero or more of the following
methods:

```javascript

/**
 * An inbound interceptor method triggered when metadata is received.
 * @param {Metadata} metadata The metadata received (response headers).
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the metadata to respond with.
 */
onReceiveMetadata(metadata, next)

/**
 * An inbound interceptor method triggered when each message is received.
 * @param {object} message The protobuf message received.
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the message to respond with.
 */
onReceiveMessage(message, next)

/**
 * An inbound interceptor method triggered when status is received.
 * @param {object} status The status received.
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the status to respond with.
 */
onReceiveStatus(status, next)
```

An `InboundInterceptorBuilder` will be provided to easily construct an inbound interceptor.

To intercept errors, implement the `onReceiveStatus` method and test for `status.code !== grpc.status.OK`.

To intercept trailers, examine `status.metadata` in the `onReceiveStatus` method.

This is a full implementation of all interceptor methods using the proposed interceptor builders:

```javascript
(new InterceptorBuilder()).withInterceptCall(function(callConstructor, options) {
    const outboundInterceptor = (new OutboundInterceptorBuilder())
        .withStart(function(metadata, next) {
            const inboundInterceptor = (new InboundInterceptorBuilder())
                .withOnReceiveMetadata(function(metadata, next) {
                    next(metadata);
                })
                .withOnReceiveMessage(function(message, next) {
                    next(message);
                })
                .withOnReceiveStatus(function(status, next) {
                    next(status);
                }).build();
            next(metadata, inboundInterceptor);
        })
        .withSendMessage(function(message, next) {
            next(messasge);
        })
        .withHalfClose(function(next) {
            next();
        })
        .withCancel(function(message, next) {
            next();
        }).build();
    const call = callConstructor(options);
    return new ForwardingCall(call, outboundInterceptor);
}).build();
```
Outbound and inbound interceptors do not need to implement all methods.

### Use

Interceptors can be configured during client construction, or on individual invocations of gRPC calls.

#### At Client Construction

An `InterceptorProvider` type will be provided for computing the association of interceptors with gRPC calls
dynamically.

```javascript
/**
* @param {Function} getInterceptorForMethod A filter method which accepts a MethodDescriptor and returns
* `undefined` (when no interceptor should be associated) or a single interceptor object.
* @constructor
*/
InterceptorProvider(getInterceptorForMethod)

```

An array of InterceptorProviders can be passed in the `options` parameter when constructing a client:
```javascript
var interceptor_providers = [
    new InterceptorProvider(function(method_descriptor) {
        if (method_descriptor.method_type === MethodType.UNARY) {
            return unary_interceptor;
        }
    }),
    new InterceptorProvider(function(method_descriptor) {
        if (method_descriptor.method_type === MethodType.SERVER_STREAMING) {
            return streaming_interceptor;
        }
    })
];
var constructor_options = {
    interceptor_providers: interceptor_providers
};
var client = new InterceptingClient('localhost:8080', credentials, constructor_options);
```

#### At Call Invocation

If an array of interceptors is provided at call invocation via `options.interceptors`, the call will ignore all
interceptors provided via the client constructor.

Example:
```javascript
client.unaryCall(message, { interceptors: [ myInterceptor ] }, callback);
```

Alternatively, an array of InterceptorProviders can be passed at call invocation (which will also supersede constructor
options):

```javascript
client.unaryCall(message, { interceptor_providers: [ myInterceptorProvider ] }, callback);
```

The framework will throw an error if both `interceptors` and `interceptor_providers` are provided in the invocation
options.

### Internal implementation

#### gRPC Operations
A new `client_interceptors` module will be added and the `getCall` method in `client.js` will be modified to use it.
The interceptor module will produce a proxy for the `grpc.Call` object which triggers any interceptors provided on the
corresponding events.

In the invocation of a `grpc.Call`'s `startBatch`, a set of gRPC opTypes are passed. This is the mapping of the
outbound opTypes to the interceptor methods they will trigger:

```
grpc.opType.SEND_INITIAL_METADATA -> start
grpc.opType.SEND_MESSAGE -> sendMessage
grpc.opType.SEND_CLOSE_FROM_CLIENT -> halfClose
```
On the inbound side, these opTypes will map to these interceptor methods when responses are received from the server:
```
grpc.opType.RECV_INITIAL_METADATA -> onReceiveMetadata
grpc.opType.RECV_MESSAGE -> onReceiveMessage
grpc.opType.RECV_STATUS_ON_CLIENT -> onReceiveStatus
```

Additional, `cancel()` and `cancelWithStatus()` are modified to trigger any `cancel()` interceptors.

#### Interceptor Order
Interceptors nest in the order provided. Providing an interceptors array like
`[interceptorA, interceptorB, interceptorC]` will produce this execution graph:
```
interceptorA outbound ->
    interceptorB outbound ->
        interceptorC outbound ->
            underlying grpc.Call ->
        interceptorC inbound ->
    interceptorB inbound ->
interceptorA inbound
```
All the appropriate interceptor methods on a given interceptor fire before the next interceptor is evaluated. (i.e.
interceptorA's `start` and `sendMessage` would fire before interceptorB's `start` and `sendMessage`)

#### Serialization and Deserialization
To provide interceptors with useful message objects, we will move all serialization and deserialization to a common
`startBatch` wrapper. This allows us to intercept the `startBatch` method and trigger outbound interceptor methods
before serialization happens. On the inbound side, we will deserialize the messages before triggering interceptor
methods.

We also box/unbox Metadata objects in the same way so interceptors get useful Metadata.

To do this requires passing certain values through the `grpc.Call` proxies via a new `context` parameter to
`startBatch`.

## Rationale

### Abstraction level
Interceptors are implemented by wrapping the `grpc.Call#startBatch` method. This level of abstraction avoids coupling
interceptors with much of the NodeJS client code since `startBatch` is the common entry point for all gRPC operations.

- If we intercepted calls at a higher level, the interceptor framework would have to account for the different types
of RPCs (unary, clientStream, serverStream, bidirectional) and could not provide the same interceptor granularity
that other gRPC language interceptors provide.
- We could intercept calls at a lower level (`call.cc`), but no additional interceptor granularity would be provided.

### Invocation-specific and client-wide interceptor configuration
Clients can be configured with interceptor providers at construction time, or interceptors can be passed to individual
call invocations. This allows configuration flexibility: a service author can provide a wrapped client constructor
which configures the interceptors as desired, and consumers of the client can override any interceptor configuration
at invocation time. If any interceptors are passed at invocation, all interceptors attached to the call during
construction are ignored.

## Implementation

The implementation will include:

- A `client_interceptors.js` module with the logic for producing a `grpc.Call` proxy.
- Modifying the `client.js` module to use the `client_interceptors` module.
- Tests which exercise the client interceptor methods for all four RPC types.
- Markdown documentation with example interceptors.

All steps will be implemented by Netflix engineers.

An implementation of server interceptors is not in scope. A separate proposal for server interceptors using similar
concepts and interfaces will be provided at a later date.
