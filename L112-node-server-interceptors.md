Node Server Interceptors
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: In Review
* Implemented in: Node.js
* Last updated: 2024-01-05
* Discussion at: https://groups.google.com/g/grpc-io/c/oQRRnPfm7Dk

## Abstract

Node.js gRPC servers will provide an interceptor interface. Operators of these servers can provide an ordered list of interceptors with methods to be executed on any inbound or outbound operation, for any incoming request. The design mirrors the client interceptors design in [L5: NodeJS Client Interceptors][L5].

## Background

Server interceptors are one of the longest-standing feature requests in the Node library, as seen in [grpc/grpc-node#419](https://github.com/grpc/grpc-node/issues/419). In addition, they are needed to implement xDS server features, particularly [A41: xDS RBAC Support][A41].


### Related Proposals:
* [L5: NodeJS Client Interceptors][L5]
* [A41: xDS RBAC Support][A41]

## Proposal

### Interceptor API

A server interceptor is a function that takes an instance of `ServerInterceptingCall` and a `ServerMethodDefinition` object, and returns another instance of that class:

```ts
function interceptor(methodDescriptor: ServerMethodDefinition, call: ServerInterceptingCall): ServerInterceptingCall {
  return new ServerInterceptingCall(call);
}
```

An interceptor function allows developers to define interception methods and to interact directly with the incoming call if necessary.

The interceptor function must return a `ServerInterceptingCall` object. A `ServerInterceptingCall` represents an element in the interceptor chain. Any intercepted methods are implemented in the `responder` object, an optional parameter to the constructor. Returning `new ServerInterceptingCall(call)` will satisfy the contract (but provide no interceptor functionality).

Note: interceptors will only be called when a request to a registered method is made.

A `ServerMethodDefinition` object describes the method that is called, and is defined as follows:

```ts
export interface ServerMethodDefinition<RequestType, ResponseType> {
  path: string;
  requestStream: boolean;
  responseStream: boolean;
  responseSerialize: Serialize<ResponseType>;
  requestDeserialize: Deserialize<RequestType>;
  originalName?: string;
}
```

A `ServerInterceptingCall` has the following API:

```ts
class ServerInterceptingCall {
  /**
   * Register the listener to handle inbound events.
   */
  start(listener: InterceptingServerListener): void;
  /**
   * Send response metadata.
   */
  sendMetadata(metadata: Metadata): void;
  /**
   * Send a response message.
   */
  sendMessage(message: any, callback: () => void): void;
  /**
   * End the call by sending this status.
   */
  sendStatus(status: PartialStatusObject): void;
  /**
   * Start a single read, eventually triggering either listener.onReceiveMessage or listener.onReceiveHalfClose.
   */
  startRead(): void;
  /**
   * Return the peer address of the client making the request, if known, or "unknown" otherwise
   */
  getPeer(): string;
  /**
   * Return the call deadline set by the client. The value is Infinity if there is no deadline.
   */
  getDeadline(): Deadline;
}
```

Most interceptors will provide a `responder` object (described below), which is passed to the `ServerInterceptingCall` constructor: `new ServerInterceptingCall(call, responder)`.

A reponder conforms to the following API:

```ts
interface Responder {
  /**
   * An interception method called before handling of an inbound call starts. Used to register the listener to handle inbound events.
   */
  start?: (next: (listener?: ServerListener) => void): void;
  /**
   * An interception method called when sending response metadata.
   */
  sendMetadata?: (metadata: Metadata, next: (metadata: Metadata) => void): void;
  /**
   * An interception method called when sending response messages.
   */
  sendMessage?: (message: any, next: (message: any) => void): void;
  /**
   * An interception method called when sending the call status.
   */
  sendStatus?: (status: PartialStatusObject, next: (status: PartialStatusObject) => void): void;
}
```

A `ResponderBuilder` will be provided to easily construct an outbound interceptor

```ts
const requester = (new ResponderBuilder())
      .withSendMetadata(function(metadata, next) {
          logger.log(metadata);
          next(metadata);
      })
      .withSendMessage(function(message, next) {
          logger.log(message);
          next(message);
      })
      .build();
```

A `listener` object implements zero or more methods for intercepting inbound operations. The listener passed into
a requester's `start` method implements all the inbound interception methods. Inbound operations will be passed through
a listener at each step in the interceptor chain. Two usage patterns are supported for listeners:
1) Pass the listener along without modification: `next(metadata, listener)`. In this case the interceptor declines
to intercept any inbound operations.
1) Create a new listener with one or more inbound interception methods and pass it to `next`. In this case the
interceptor will fire on the inbound operations implemented in the new listener.

A listener is an object that conforms to the following API:

```ts
interface ServerListener {
  /**
   * An inbound interception method called when metadata is received. After this event goes through the interception stack, the regular method handler is called.
   */
  onReceiveMetadata?: (metadata: Metadata, next: (metadata: Metadata) => void): void;
  /**
   * An inbound interception method called when each message is received.
   */
  onReceiveMessage?: (message: any, next: (message: any) => void): void;
  /**
   * An inbound interception method called when the inbound message stream ends.
   */
  onReceiveHalfClose?: (next: () => void): void;
  /**
   * An inbound interception method called when the request ends for any reason, including when cancelled by the remote end.
   */
  onCancel?: (): void;
}
```

A `ServerListenerBuilder` will be provided to easily construct a listener:

```ts
const listener = (new ServerListenerBuilder())
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

An `InterceptingServerListener` is an object like a `ServerListener` but without the `next` callbacks:

```ts
export interface InterceptingServerListener {
  onReceiveMetadata(metadata: Metadata): void;
  onReceiveMessage(message: any): void;
  onReceiveHalfClose(): void;
  onCancel(): void;
}
```

#### Correspondence with client interceptor APIs

The server interceptor APIs are roughly mirrors of the client interceptor APIs, in the sense that send operations on the client correspond to receive operations on the server, and vice versa. In particular, the methods are matched this way as follows:

| Client API | Server API | Notes |
| ---------- | ---------- | ----- |
| `Requester.start` | `ServerListener.onReceiveMetadata` | |
| `Requester.sendMessage` | `ServerListener.onReceiveMessage` | |
| `Requester.halfClose` | `ServerListener.onReceiveHalfClose` | |
| `Requester.cancel` | `ServerListener.onCancel` | This server API also handles calls ending in ways other than client cancellation. |
| | `Responder.start` | This is a utility function for setting up interceptors that does not correspond to any client action. |
| `Listener.onReceiveMetadata` | `Responder.sendMetadata` | |
| `Listener.onReceiveMessage` | `Responder.sendMessage` | |
| `Listener.onReceiveStatus` | `Responder.sendStatus` | |

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

### Use

The `Server` constructor's `options` parameter will accept a new option `interceptors`, which is an array of interceptor functions.


#### Interceptor nesting

Interceptors nest in the order provided. Providing an interceptors array like `[interceptorA, interceptorB, interceptorC]` will produce this execution graph:

```
underlying ServerCall created ->
  interceptorA call ->
    interceptorB call ->
      interceptorC call ->
        start called ->
      interceptorC start ->
    interceptorB start ->
  interceptorA start ->
underlying ServerCall start ->
  interceptorA onReceiveMetadata ->
    interceptorB onReceiveMetadata ->
      interceptorC onReceiveMetadata ->
        method handler invocation ->
      interceptorC outbound ->
    interceptorB outbound ->
  interceptorA outbound ->
underlying ServerCall ->
  interceptorA inbound ->
    interceptorB inbound ->
      interceptorC inbound ->
        method handler
```

## Rationale

### Interception API

The `Responder` and `ServerListener` APIs are modeled very closely on the `Requester` and `Listener` APIs in the [Client Interceptors design][L5], with small modifications to account for the differences in events that are asymmetrical between the client and the server. This should make it easy for developers who are already familiar with the client interceptor API to create server interceptors.

#### The `start` method

The `Responder#start` method is the only one that does not correspond to any actual network activity. At the HTTP/2 level, the beginning of an incoming call is signalled by receiving the request headers for that call. However, this start method provides a simple way to register the listeners in the proper order after all interceptors have been invoked and to allow those listeners to handle the incoming metadata.

Unlike in the corresponding method in the client interceptor's `Requester`, this `start` method does not take any argument other than the `next` callback. On the client having direct access to the previous listener is useful to be able to fake or copy incoming network events. For example, a client-side caching interceptor could pass a previously received message directly to the listener for a new request, without any corresponding network event on the current request. There are not similar use cases on the server, so the corresponding argument is omitted.

#### The `onCancel` method

The `ServerListener#onCancel` method is "called when the request ends for any reason, including when cancelled by the remote end", which does not strictly match the meaning of the term "cancel". The purpose of handling cancellation on the server is to let the handler know when it does not need to do any further processing to send information to the client, because the client will not see it. So, it is useful to notify whenever that condition is true, no matter what triggered it.

The `onCancel` method has no callback because its purpose is to notify each interceptor and the handler that the call has ended. There is no use in allowing one interceptor to choose not to propagate that notification or to delay propagating it.

### Interceptors only called for registered methods

For consistency with the client interceptors design, interceptors will see un-serialized message objects. The deserialization functions are only known for registered methods, so interceptors can only be called for registered methods. There is space for future APIs to fill that gap without breaking anything.

### Single injection point

The client interceptor design has interceptor lists and interceptor provider lists, and interceptors specified at client construction vs interceptors specified for individual call invocations. In contrast, this design only specifies that an interceptor list can be provided at server construction. Per-call interceptors simply don't make sense on the server side, because there is no reasonable point at which to inject them that wouldn't just act as a regular interceptor. Interceptor providers on the other hand could work on the server, but they are not necessary for basic interceptor functionality. The primary purpose of this design is to solidify the interceptor part of the design. Additional injection points for interceptors could be specified in the future, and the way they interact with each other can be decided at that time.

## Implementation

I (murgatroid99) will implement this immediately.

[L5]: https://github.com/grpc/proposal/blob/master/L5-node-client-interceptors.md
[A41]: https://github.com/grpc/proposal/blob/master/A41-xds-rbac.md
