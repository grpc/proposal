NodeJS Client Interceptors
----
* Author(s): David Vroom-Duke (dduke@netflix.com), William Thurston (wthurston@netflix.com), David Liu (dliu@netflix.com), Howard Yuan (hyuan@netflix.com), Eran Landau (elandau@netflix.com)
* Approver: a11r
* Status: Draft
* Implemented in: NodeJS
* Last updated: 2017-03-09
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/LxT1JjN33Q4

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
 * @param {object} options
 * @param {MethodDescriptor} options.method_descriptor A container of properties describing the call (see below)
 * @param {grpc.CallCredentials} [options.credentials] Credentials used to authenticate with the server
 * @param {number} [options.deadline] The deadline for the call
 * @param {string} [options.host] The server to call
 * @param {grpc.Call} [options.parent] The parent call
 * @param {number} [options.propagate_flags] Propagation flags for the underlying grpc.Call
 * @return {InterceptingCall}
 */
interceptCall(options)
```

The `interceptCall` method allows developers to modify the call options, store any call-scoped values needed, and define
interceptor methods. An `InterceptorBuilder` will be provided to easily construct an interceptor.

`interceptCall` must return an `InterceptingCall` object. An `InterceptingCall` is a container for the call options
and an optional `caller` object. Returning `new InterceptingCall(options)` will satisfy the contract (but provide no
interceptor functionality).

Most interceptors will define a `caller` object (described below) which is passed to the the `InterceptingCall`
constructor: `return new InterceptingCall(options, caller)`

A simple interceptor example:
```javascript
var interceptor = (new InterceptorBuilder())
    .withInterceptCall(function(options) {
        var caller = (new CallerBuilder())
            .withSendMessage(function(message, next) {
                logger.log(message);
                next(message);
            })
            .build();
        return new InterceptingCall(options, caller);
    })
    .build();
```

The `options` argument to the `InterceptingCall` constructor includes all the options accepted by the base `grpc.Call`
constructor. Modifying the options that are passed to the `InterceptingCall` constructor will have the effect of
changing the options passed to the underlying `grpc.Call` constructor.

Additionally, the `options` argument includes a `MethodDescriptor` object. The `MethodDescriptor` is a container for
properties of the call which are used internally and may also be useful to interceptors:

```javascript
/**
 * @param {string} name The name of the call, i.e. 'myCall'
 * @param {string} path The full path of the call, i.e. '/MyService/MyCall'
 * @param {MethodType} method_type One of four types:
 *     MethodType.UNARY,
 *     MethodType.CLIENT_STREAMING,
 *     MethodType.SERVER_STREAMING, or
 *     MethodType.BIDI_STREAMING
 * @param {Function} serialize The function used to serialize a message
 * @param {Function} deserialize The function used to deserialize a message
 * @constructor
 */
MethodDescriptor(name, path, method_type, serialize, deserialize)
```

*Do not modify the `options.method_descriptor` object, it is not used by the underlying gRPC code and will only affect
downstream interceptors*

A `caller` object is a POJO implementing zero or more outbound interceptor methods:

```javascript
/**
 * An interceptor method called before an outbound call has started.
 * @param {Metadata} metadata The call's outbound metadata (request headers).
 * @param {object} listener The listener which will be intercepting inbound operations
 * @param {Function} next A callback which continues the gRPC interceptor chain.
 * next takes two arguments: A Metadata object and a listener
 */
start(metadata, listener, next)
```
```javascript
/**
 * An interceptor method called prior to every outbound message.
 * @param {object} A protobuf message
 * @param {function} next A callback which continues the gRPC interceptor chain, called with the message to send.
 */
sendMessage(message, next)
```
```javascript
/**
 * An interceptor method called when the outbound stream closes (after the message is sent).
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
halfClose(next)
```
```javascript
/**
 * An interceptor method called when the stream is canceled from the client.
 * @param {message} string|null A cancel message if provided
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
cancel(message, next)
```

A `CallerBuilder` will be provided to easily construct an outbound interceptor:

```javascript
var caller = (new CallerBuilder())
    .withStart(function(metadata, listener, next) {
        logger.log(metadata);
        next(metadata, listener);
    })
    .withSendMessage(function(message, next) {
        logger.log(message);
        next(message);
    })
    .build();
```

A `listener` object implements zero or more methods for intercepting inbound operations. The listener passed into
a caller's `start` method implements all the inbound interceptor methods and represents the listener chain constructed
by any previous interceptors. Three usage patterns are supported for listeners:
1) Pass the listener along without modification: `next(metadata, listener)`. In this case the interceptor declines
to intercept any inbound operations.
2) Create a new listener with one or more inbound interceptor methods and pass it to `next`. In this case the
interceptor will fire on the inbound operations implemented in the new listener.
3) Store the listener to make direct inbound calls on later. This effectively short-circuits the interceptor stack.
An example of an interceptor using this pattern to provide client-side caching is included below.

*Do not modify the listener passed in. Either pass it along unmodified or call methods on it to short-circuit the
interceptor stack.*

The `listener` methods are:
```javascript

/**
 * An inbound interceptor method triggered when metadata is received.
 * @param {Metadata} metadata The metadata received (response headers).
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the metadata to respond with.
 */
onReceiveMetadata(metadata, next)
```
```javascript
/**
 * An inbound interceptor method triggered when each message is received.
 * @param {object} message The protobuf message received.
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the message to respond with.
 */
onReceiveMessage(message, next)
```
```javascript
/**
 * An inbound interceptor method triggered when status is received.
 * @param {object} status The status received.
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the status to respond with.
 */
onReceiveStatus(status, next)
```

A `ListenerBuilder` will be provided to easily construct a listener:

```javascript
var listener = (new ListenerBuilder())
    .withOnReceiveMetadata(function(metadata, next) {
        logger.log(metadata);
        next(metadata);
    })
    .withOnReceiveMessage(function(message, next) {
        logger.log(message);
        next(message);
    })
    .build();
```

A `StatusBuilder` will be provided to produce gRPC status objects:

```javascript
var status = (new StatusBuilder())
    .withCode(grpc.status.OK)
    .withDetails('Status message')
    .withMetadata(new Metadata())
    .build();
```

**To intercept errors, implement the `onReceiveStatus` method and test for `status.code !== grpc.status.OK`.**

**To intercept trailers, examine `status.metadata` in the `onReceiveStatus` method.**


### Examples
#### Simple
A trivial implementation of all interceptor methods using the builders:
```javascript
var interceptor = (new InterceptorBuilder())
    .withInterceptCall(function(options) {
        const caller = (new CallerBuilder())
            .withStart(function(metadata, listener, next) {
                const newListener = (new ListenerBuilder())
                    .withOnReceiveMetadata(function(metadata, next) {
                        next(metadata);
                    })
                    .withOnReceiveMessage(function(message, next) {
                        next(message);
                    })
                    .withOnReceiveStatus(function(status, next) {
                        next(status);
                    })
                    .build();
                next(metadata, newListener);
            })
            .withSendMessage(function(message, next) {
                next(messasge);
            })
            .withHalfClose(function(next) {
                next();
            })
            .withCancel(function(message, next) {
                next();
            })
            .build();
        return new InterceptingCall(options, caller);
    })
    .build();
```
(Callers and listeners do not need to implement all methods)

#### Advanced
These advanced examples are specific to certain types of RPC calls (unary, client-streaming, etc), because interceptors
operate on gRPC batches. Any given interceptor must work for every batch in the RPC calls it applies to. Since different
types of RPC calls construct batches with different sets of operations, interceptors which delay or short-circuit
operations will run into batch boundaries for some RPC call types. The limitations should be intuitive, for example: a
caching interceptor would not make sense for a streaming RPC given that inbound messages are not associated with
outbound messages.

**Caching**

An example of a caching interceptor for unary RPCs which stores the provided listener for later use (short-circuiting
the call if there is a cache hit):
```javascript
// Unary RPCs only
var interceptor = (new InterceptorBuilder())
    .withInterceptCall(function(options) {
        var savedMetadata;
        var startNext;
        var savedListener;
        var savedMessage;
        var messageNext;
        var caller = (new CallerBuilder())
            .withStart(function(metadata, listener, next) {
                savedMetadata = metadata;
                savedListener = listener;
                startNext = next;
            })
            .withSendMessage(function(message, next) {
                savedMessage = message;
                messageNext = next;
            })
            .withHalfClose(function(next) {
                var cachedValue = _getCachedResponse(savedMessage.value);
                if (cachedValue) {
                    var cachedMessage = new Message(cachedValue);
                    savedListener.onReceiveMetadata(new Metadata());
                    savedListener.onReceiveMessage(cachedMessage);
                    savedListener.onReceiveStatus((new StatusBuilder())
                        .withCode(grpc.status.OK)
                        .build());
                } else {
                    registry.addCall(savedMessage.value + '_miss');
                    var newListener = (new ListenerBuilder())
                        .withOnReceiveMessage(function(message, next) {
                            _store(savedMessage.value, message.value);
                            next(message);
                        })
                        .build();
                    startNext(savedMetadata, newListener);
                    messageNext(savedMessage);
                    next();
                }
            })
            .build();
        return new InterceptingCall(options, caller);
    })
    .build();
```

**Retries**

An example retry interceptor for unary RPCs creates new calls when the status shows a failure:
```javascript
// Unary RPCs only
var maxRetries = 3;
var interceptor = (new InterceptorBuilder())
    .withInterceptCall(function(options) {
        var savedMetadata;
        var savedSendMessage;
        var savedReceiveMessage;
        var savedMessageNext;
        var methodName = options.method_descriptor.name;
        var caller = (new CallerBuilder())
            .withStart(function(metadata, listener, next) {
                savedMetadata = metadata;
                var new_listener = (new ListenerBuilder())
                    .withOnReceiveMessage(function(message, next) {
                        savedReceiveMessage = message;
                        savedMessageNext = next;
                    })
                    .withOnReceiveStatus(function(status, next) {
                        var retries = 0;
                        var retry = function(message, metadata, options) {
                            retries++;
                            client[methodName](message, metadata, options, function(err, response) {
                                if (err) {
                                    if (retries <= maxRetries) {
                                        retry(message, metadata, options);
                                    } else {
                                        savedMessageNext(response);
                                        next(status);
                                    }
                                    return;
                                }
                                var ok_status = (new StatusBuilder())
                                    .withCode(grpc.status.OK)
                                    .build();
                                savedMessageNext(response);
                                next(ok_status);
                            });
                        };
                        if (status.code !== grpc.status.OK) {
                            var new_options = _.clone(options);
                            new_options.interceptors = [];
                            retry(savedSendMessage, savedMetadata, new_options);
                        } else {
                            savedMessageNext(savedReceiveMessage);
                            next(status);
                        }
                    })
                    .build();
                next(metadata, new_listener);
            })
            .withSendMessage(function(message, next) {
                savedSendMessage = message;
                next(message);
            })
            .build();
        return new InterceptingCall(options, caller);
    })
    .build();
```

**Fallbacks**

An example of providing fallbacks to failed requests for unary or client-streaming RPCs:
```javascript
// Unary or client-streaming RPCs only
var interceptor = (new InterceptorBuilder())
    .withInterceptCall(function(options) {
        var savedMessage;
        var savedMessageNext;
        var caller = (new CallerBuilder())
            .withStart(function(metadata, listener, next) {
                var new_listener = (new ListenerBuilder())
                    .withOnReceiveMessage(function(message, next) {
                        savedMessage = message;
                        savedMessageNext = next;
                    })
                    .withOnReceiveStatus(function(status, next) {
                        if (status.code !== grpc.status.OK) {
                            savedMessageNext(fallback_response);
                            next((new StatusBuilder())
                                .withCode(grpc.status.OK)
                                .build());
                        } else {
                            savedMessageNext(savedMessage);
                            next(status);
                        }
                    })
                    .build();
                next(metadata, new_listener);
            })
            .build();
        return new InterceptingCall(options, caller);
    })
    .build();
```

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

The order of the InterceptorProviders will determine the order of the resulting interceptor stack.

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

#### Interceptor nesting
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

#### Horizontal execution
The order of execution for interceptor methods is by operation. Given a chain of interceptors and a batch of operations,
we will execute all interceptor methods for the current operation before moving to the next. For example if you
define three interceptors which all implement `start` and `sendMessage` methods, the execution order would be:
```
interceptorA start
interceptorB start
interceptorC start
interceptorA sendMessage
interceptorB sendMessage
interceptorC sendMessage
```

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
