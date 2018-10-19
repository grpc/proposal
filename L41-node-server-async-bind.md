Asynchronous port binding method in Node Server API
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node
* Last updated: 28-09-2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/Uio1Ja6GRHI

## Abstract

Add a method to the Node gRPC `Server` class that binds a port as an asynchronous operation.

## Background

In the initial development of the Node gRPC library, the `Server` class method to bind a port, called `bind`, was implemented as a synchronous operation because it was available as a synchronous operation in the underlying core library. In contrast, in the Node built in `net` module, the `Server` class's `listen` method, which performs a similar function, is asynchronous. In the new pure JavaScript Node gRPC library, the underlying network operations use the `net` module, so it is impossible to implement the `bind` method as a synchronous operation.


## Proposal

We would add a method called `bindAsync` to the `Server` class that implements the same functionality as `bind`, but provides the result asynchronously. The specific function signature would be as follows:

```ts
class Server {

  // ...

  /**
   * @param port The port that the server should bind on, in the format "address:port"
   * @param creds Server credentials object to be used for SSL.
   * @param callback Callback to be called asynchronously with the result of binding the port
   */
  bindAsync(port: string, creds: ServerCredentials, callback: (error: Error, port: number)=>void): void;
}
```

The semantics of the arguments are exactly the same as with the existing `bind` method, and the semantics of the port passed to the callback are identical to the semantics of the return value of the existing `bind` method: a negative number indicates failure, and a positive number indicates success binding to that port number. An error will also be passed to the callback if `port` is negative.

## Rationale

The semantics of `bindAsync` would be identical to the semantics of `bind` except it is asynchronous, so the name communicates that. Keeping the semantics and API as similar as possible minimizes the work need to transition between the two. The semantically redundant `error` is passed to the callback for greater consistency with other Node APIs, so that it can be used with functions such as `util.promisify()`. We could also implement the asynchrony using promises, but a callback based API is more consistent with the rest of the existing API. In the future, promise support could be added, perhaps by returning a promise when no callback is provided.


## Implementation

I (@murgatroid99) will implement this in the grpc library as soon as this proposal is accepted, and in the @grpc/grpc-js library as part of the server implementation.
