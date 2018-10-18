NodeJS Client Interceptors
----
* Author(s): David Vroom-Duke (dduke@netflix.com), William Thurston (wthurston@netflix.com), David Liu (dliu@netflix.com), Howard Yuan (hyuan@netflix.com), Eran Landau (elandau@netflix.com)
* Approver: murgatroid99
* Status: Final
* Implemented in: NodeJS
* Last updated: 2017-06-01
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

### A simple example
This example logs all outbound messages:
```javascript
var interceptor = function(options, nextCall) {
    return new InterceptingCall(nextCall(options), {
        sendMessage: function(message, next) {
            console.log(message);
            next(message);
        }
    });
};
client.myCall(message, { interceptors: [interceptor] });
```

### Interceptor API
An interceptor is a function which takes an `options` object and a `nextCall` function and returns an
`InterceptingCall` object:

```javascript
/**
 * @param {object} options
 * @param {MethodDescriptor} options.method_descriptor A container of properties describing the call (see below)
 * @param {grpc.CallCredentials} [options.credentials] Credentials used to authenticate with the server
 * @param {number} [options.deadline] The deadline for the call
 * @param {string} [options.host] The server to call
 * @param {grpc.Call} [options.parent] The parent call
 * @param {number} [options.propagate_flags] Propagation flags for the underlying grpc.Call
 * @param {function} nextCall Constructs the next interceptor in the chain
 * @return {InterceptingCall}
 */
var interceptor = function(options, nextCall) {
    return new InterceptingCall(nextCall(options));
};
```

An interceptor function allows developers to modify the call options, store any call-scoped values needed, and define
interception methods.

The interceptor function must return an `InterceptingCall` object. An `InterceptingCall` represents an element in the
interceptor chain. Any intercepted methods are implemented in the `requester` object, an optional parameter to the
constructor. Returning `new InterceptingCall(nextCall(options))` will satisfy the contract (but provide no interceptor
functionality).

Most interceptors will define a `requester` object (described below) which is passed to the the `InterceptingCall`
constructor: `return new InterceptingCall(nextCall(options), requester)`

The `options` argument to the `nextCall` function includes all the options accepted by the base `grpc.Call`
constructor. Modifying the options that are passed to the `nextCall` function will have the effect of
changing the options passed to the underlying `grpc.Call` constructor.

Additionally, the `options` argument includes a `MethodDescriptor` object. The `MethodDescriptor` is a container for
properties of the call which are used internally and may also be useful to interceptors:

```javascript
/**
 * @param {string} name The name of the call, i.e. 'myCall'
 * @param {string} service_name The name of the service
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
MethodDescriptor(name, service_name, path, method_type, serialize, deserialize)
```

*Do not modify the `options.method_descriptor` object. The MethodDescriptor passed to the interceptor options is not
consumed by the underlying gRPC code and changes will only affect downstream interceptors*

A `requester` object is a plain Javascript object implementing zero or more outbound interception methods:

```javascript
/**
 * An interception method called before an outbound call has started.
 * @param {Metadata} metadata The call's outbound metadata (request headers).
 * @param {object} listener The listener which will be intercepting inbound operations
 * @param {Function} next A callback which continues the gRPC interceptor chain.
 * next takes two arguments: A Metadata object and a listener
 */
start(metadata, listener, next)
```
```javascript
/**
 * An interception method called prior to every outbound message.
 * @param {object} A protobuf message
 * @param {function} next A callback which continues the gRPC interceptor chain, called with the message to send.
 */
sendMessage(message, next)
```
```javascript
/**
 * An interception method called when the outbound stream closes (after the message is sent).
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
halfClose(next)
```
```javascript
/**
 * An interception method called when the stream is canceled from the client.
 * @param {message} string|null A cancel message if provided
 * @param {function} next A callback which continues the gRPC interceptor chain.
 */
cancel(message, next)
```

A `RequesterBuilder` will be provided to easily construct an outbound interceptor:

```javascript
var requester = (new RequesterBuilder())
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
a requester's `start` method implements all the inbound interception methods. Inbound operations will be passed through
a listener at each step in the interceptor chain. Three usage patterns are supported for listeners:
1) Pass the listener along without modification: `next(metadata, listener)`. In this case the interceptor declines
to intercept any inbound operations.
2) Create a new listener with one or more inbound interception methods and pass it to `next`. In this case the
interceptor will fire on the inbound operations implemented in the new listener.
3) Store the listener to make direct inbound calls on later. This effectively short-circuits the interceptor stack.
An example of an interceptor using this pattern to provide client-side caching is included below in the Examples
section. Short-circuiting a request in this way will skip interceptors which have not yet fired, but will fire the
listeners on any 'earlier' interceptors in the stack.

*Do not modify the listener passed in. Either pass it along unmodified or call methods on it to short-circuit the
interceptor stack.*

The `listener` methods are:
```javascript

/**
 * An inbound interception method triggered when metadata is received.
 * @param {Metadata} metadata The metadata received (response headers).
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the metadata to respond with.
 */
onReceiveMetadata(metadata, next)
```
```javascript
/**
 * An inbound interception method triggered when each message is received.
 * @param {object} message The protobuf message received.
 * @param {function} next A callback which continues the gRPC interceptor chain. Pass it the message to respond with.
 */
onReceiveMessage(message, next)
```
```javascript
/**
 * An inbound interception method triggered when status is received.
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

#### Internal Errors and Exceptions

Exceptions are not handled by the interceptor framework, we expect
interceptor authors to either handle the exceptions their interceptors
can throw or assume the Node process will crash on exceptions. This is in
accordance with the [Joyent guidance](https://www.joyent.com/node-js/production/design/errors)
to only catch exceptions which are known to be operational errors.

In the case of an error during the execution of an interceptor,
interceptor authors have the following options:
1) Short-circuiting the call and returning a gRPC error to the client's consumer.
2) Throwing an exception which the client's consumer will have to catch
or crash on.
3) Doing nothing and continuing with the call.


### Examples
#### Simple
A trivial implementation of all interception methods without using the builders:
```javascript
var interceptor = function(options, nextCall) {
    var requester = {
        start: function(metadata, listener, next) {
            var newListener = {
                onReceiveMetadata: function(metadata, next) {
                    next(metadata);
                },
                onReceiveMessage: function(message, next) {
                    next(message);
                },
                onReceiveStatus: function(status, next) {
                    next(status);
                }
            };
            next(metadata, newListener);
        },
        sendMessage: function(message, next) {
            next(messasge);
        },
        halfClose: function(next) {
            next();
        },
        cancel: function(message, next) {
            next();
        }
    };
    return new InterceptingCall(nextCall(options), requester);
};
```
(Requesters and listeners do not need to implement all methods)

#### Advanced Examples
These advanced examples are specific to certain types of RPC calls (unary, client-streaming, etc). Using interceptors
to provide advanced behavior such as delays, retries and caching will not apply to all RPC types. The limitations should
be intuitive, for example: a caching interceptor would not make sense for a bidi streaming RPC given that inbound
messages are not associated with outbound messages.

**Caching**

An example of a caching interceptor for unary RPCs which stores the provided listener for later use (short-circuiting
the call if there is a cache hit):
```javascript
// Unary RPCs only
var interceptor = function(options, nextCall) {
    var savedMetadata;
    var startNext;
    var savedListener;
    var savedMessage;
    var messageNext;
    var requester = {
        start: function(metadata, listener, next) {
            savedMetadata = metadata;
            savedListener = listener;
            startNext = next;
        },
        sendMessage: function(message, next) {
            savedMessage = message;
            messageNext = next;
        },
        halfClose: function(next) {
            var cachedValue = _getCachedResponse(savedMessage.value);
            if (cachedValue) {
                var cachedMessage = new Message(cachedValue);
                savedListener.onReceiveMetadata(new Metadata());
                savedListener.onReceiveMessage(cachedMessage);
                savedListener.onReceiveStatus({code: grpc.status.OK});
            } else {
                var newListener = {
                    onReceiveMessage: function(message, next) {
                        _store(savedMessage.value, message.value);
                        next(message);
                    }
                };
                startNext(savedMetadata, newListener);
                messageNext(savedMessage);
                next();
            }
        }
    };
    return new InterceptingCall(nextCall(options), requester);
};
```

**Retries**

An example retry interceptor for unary RPCs creates new calls when the status shows a failure:
```javascript
// Unary RPCs only
var maxRetries = 3;
var interceptor = function(options, nextCall) {
    var savedMetadata;
    var savedSendMessage;
    var savedReceiveMessage;
    var savedMessageNext;
    var requester = {
        start: function(metadata, listener, next) {
            savedMetadata = metadata;
            var newListener = {
                onReceiveMessage: function(message, next) {
                    savedReceiveMessage = message;
                    savedMessageNext = next;
                },
                onReceiveStatus: function(status, next) {
                    var retries = 0;
                    var retry = function(message, metadata) {
                        retries++;
                        var newCall = nextCall(options);
                        newCall.start(metadata, {
                            onReceiveMessage: function(message) {
                                savedReceiveMessage = message;
                            },
                            onReceiveStatus: function(status) {
                                if (status.code !== grpc.status.OK) {
                                    if (retries <= maxRetries) {
                                        retry(message, metadata);
                                    } else {
                                        savedMessageNext(savedReceiveMessage);
                                        next(status);
                                    }
                                } else {
                                    savedMessageNext(savedReceiveMessage);
                                    next({code: grpc.status.OK});
                                }
                            }
                        })
                    };
                    if (status.code !== grpc.status.OK) {
                        retry(savedSendMessage, savedMetadata);
                    } else {
                        savedMessageNext(savedReceiveMessage);
                        next(status);
                    }
                }
            };
            next(metadata, newListener);
        },
        sendMessage: function(message, next) {
            savedSendMessage = message;
            next(message);
        }
    };
    return new InterceptingCall(nextCall(options), requester);
};
```

**Fallbacks**

An example of providing fallbacks to failed requests for unary or client-streaming RPCs:
```javascript
// Unary or client-streaming RPCs only
var fallbackResponse = new Message('fallback');
var interceptor = function(options, nextCall) {
    var savedMessage;
    var savedMessageNext;
    var requester = {
        start: function(metadata, listener, next) {
            var new_listener = {
                onReceiveMessage: function(message, next) {
                    savedMessage = message;
                    savedMessageNext = next;
                },
                onReceiveStatus: function(status, next) {
                    if (status.code !== grpc.status.OK) {
                        savedMessageNext(fallbackResponse);
                        next({node: grpc.status.OK});
                    } else {
                        savedMessageNext(savedMessage);
                        next(status);
                    }
                }
            };
            next(metadata, new_listener);
        }
    };
    return new InterceptingCall(nextCall(options), requester);
};
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
To intercept operations at the call level, two major changes to the gRPC NodeJS client implementation are required:
1) Split out the logic for calling `startBatch` and for handling the results of `startBatch`. Interceptor chains for
each operation need to run on the outbound path and the inbound batch before this logic.

2) Wrap each call with the logic for assembling interceptor chains and executing them.

For 1), we will add a `client_batches` module which will enumerate each distinct 'batch type' used by gRPC clients. The
batching logic currently in the `makeUnaryRequest` and similar functions will be moved to the `client_batches` module,
with methods for wiring a call instance up to the batch logic and the interceptor chains.

For 2), we will add a `client_interceptors` module with methods for building interceptor chains and triggering
underlying calls.

The `requester` and `listener` methods correspond to gRPC batch operations. This is the mapping of the
outbound opTypes to the `requester` methods they will trigger:

```
grpc.opType.SEND_INITIAL_METADATA -> start
grpc.opType.SEND_MESSAGE -> sendMessage
grpc.opType.SEND_CLOSE_FROM_CLIENT -> halfClose
```
On the inbound side, these opTypes will map to these `listener` methods when responses are received from the server:
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

#### Horizontal execution
The order of execution for interception methods is by operation. Given a chain of interceptors and a batch of operations,
we will execute all interception methods for one operation before moving to the next. For example if you
define three interceptors which all implement `start` and `sendMessage` methods, the execution order would be:
```
interceptorA start
interceptorB start
interceptorC start
interceptorA sendMessage
interceptorB sendMessage
interceptorC sendMessage
```

#### Batch Types
There are 8 distinct batch types in the NodeJS client implementation, and each RPC type uses some combination of those
8 batch types:
- Unary (all operations)
- Metadata only
- Close only
- Sending streaming message
- Receive streaming message
- Receive status
- Send synchronous message
- Receive synchronous message

This proposal moves the logic for these batch types out of the RPC functions to reduce code duplication and provide
a common mechanism for intercepting each batch type. The execution order for a metadata-only batch with 3 interceptors
will look like this:
##### Outbound
```
Consumer makes client call
Interceptor A start
Interceptor B start
Interceptor C start
Outbound batch logic:
  Convert metadata to internal representation
  Call startBatch with metadata operations
```
##### Inbound
```
Metadata received from server
Convert internal representation to Metadata object
Interceptor C Receive Metadata
Interceptor B Receive Metadata
Interceptor A Receive Metadata
Inbound batch logic:
  Emit resulting metadata
```

#### Initialization
Interceptors are initialized at each gRPC call invocation. A call which runs multiple batches will reuse the same
interceptors across all batches. A second invocation of the gRPC call will initialize new interceptors. Interceptors
can use this to store call-scoped data in their initialization function.

## Rationale

### Abstraction level
Interceptors are call-scoped so interceptor authors will be able to reason about the initialization of their
interceptors easily. This requires more changes to client.js than intercepting batches individually, but makes it easier
for interceptors to operate on different RPC types without special cases.

### Invocation-specific and client-wide interceptor configuration
Clients can be configured with interceptor providers at construction time, or interceptors can be passed to individual
call invocations. This allows configuration flexibility: a service author can provide a wrapped client constructor
which configures the interceptors as desired, and consumers of the client can override any interceptor configuration
at invocation time. If any interceptors are passed at invocation, all interceptors attached to the call during
construction are ignored.

## Implementation

The implementation will include:

- A `client_interceptors.js` module with the logic for constructing and executing interceptor chains.
- A `client_batches.js` module to define the batch types and provide a common mechanism to intercept them.
- Modifying the `client.js` module to use the new modules.
- Tests which exercise the client interception methods for all four RPC types.
- Markdown documentation with example interceptors.

All steps will be implemented by Netflix engineers.

An implementation of server interceptors is not in scope. A separate proposal for server interceptors using similar
concepts and interfaces will be provided at a later date.
