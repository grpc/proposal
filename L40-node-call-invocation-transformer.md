gRPC Node Call Invocation Transformer API
----
* Author(s): murgatroid99, WeiranFang
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node
* Last updated: 26-09-2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/f3v4SBvj7L4

## Abstract

Add a client API for transforming some paramters and intermediate variables for each call made using that client.

## Background

Some advanced call interception use cases require functionality that is not provided by the existing `CallInterceptor` API. 


### Related Proposals: 
* [L5-NODEJS-CLIENT_INTERCEPTORS](L5-NODEJS-CLIENT_INTERCEPTORS.md)

## Proposal

Define the type `CallProperties` as follows:

```ts
interface CallProperties {
  // The argument to the method. Only available for unary and client-streaming methods
  argument: any;
  // The metadata that will be sent to that data
  metadtata: Metadata;
  // The call object that will be returned by the method
  call: ClientUnaryCall | ClientReadableStream | ClientWritableStream | ClientDuplexStream;
  // The channel object that will be used to transmit the request
  channel: Channel;
  // An object describing the request method
  methodDefinition: MethodDefinition;
  // The options object passed to the call
  callOptions: CallOptions;
  // The callback passed to the method. Only available for unary and client-streaming methods
  callback: requestCallback;
}
```

Add a new option to the `Client` constructor `options` parameter called `callInvocationTransformer`, that accepts a function that takes a `CallProperties` object as an input and returns another `CallProperties` object as the output. This function can read and modify any of the values in the object, and the returned values will be used for processing the request, with the caveat that some modifications may cause the request to be processed improperly.

## Rationale

The specific use case that prompted this proposal needs change the channel based on the unserialized argument, and to track response messages and call start and end associated with each channel. The only part of the code with access to all of those things is the beginning of each call invocation function in the client. So, for generality, this API provides access to (almost) all of the objects used to invoke calls and allows for them to be modified at the beginning of call invocation.


## Implementation

The implementation in the `grpc` library will be completed by @WeiranFang after this proposal is accepted. An initial implementation PR exists at grpc/grpc-node#557.

The implementation in the `@grpc/grpc-js` library will be completed by murgatroid99 after the client interceptors API is implemented in that library.
