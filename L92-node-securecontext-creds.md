L92: Add a new Node API to create credentials from a SecureContext
----
* Author(s): murgatroid99
* Approver: wenbozhu
* Implemented in: Node.js (grpc-js)
* Last updated: 2021-12-10
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add a new channel credentials creation API to grpc-js that uses the `SecureContext` type defined in Node's built in TLS module.

## Background

The existing `credentials.createSsl` API in the Node library handles a specific set of basic parameters based on the parameters handled by the corresponding API in the gRPC core library. Some Node users have requested the ability to use other parameters that Node's TLS APIs support, but gRPC does not. In particular:

 - [grpc/grpc-node#1712](https://github.com/grpc/grpc-node/issues/1712): The user wants to be able to pass in a passphrase to handle encrypted keys.
 - [grpc/grpc-node#1802](https://github.com/grpc/grpc-node/issues/1802): The user is apparently able to solve their problem by changing the TLS version and cipher list.
 - [grpc/grpc-node#1977](https://github.com/grpc/grpc-node/issues/1977): The user wants to pass the private key and certificate chain combined in the PFX format.

## Proposal

The Node grpc-js library will add the function `credentials.createFromSecureContext`, which will take as arguments a [`SecureContext`](https://nodejs.org/api/tls.html#tlscreatesecurecontextoptions) and an optional `VerifyOptions` object (the final parameter of the existing `credentials.createSsl`). gRPC has a standard chiper list, but the cipher list is part of the secure context, so these credentials objects will use Node's default cipher list instead of gRPC's default cipher list. For the same reason, these credentials will use Node's default root certs list instead of gRPC's.

The following two calls are almost equivlaent, other than the cipher list, demonstrating the correspondence between the existing and new APIs:

```ts
// Existing API
credentials.createSsl(rootCerts, privateKey, certChain, verifyOptions);

// New API
credentials.createFromSecureContext(tls.createSecureContext({
  ca: rootCerts,
  key: privateKey,
  cert: certChain
}), verifyOptions);
```

## Rationale

Internally, grpc-js's secure channel credentials implementation uses the built in Node TLS APIs, but access to those features is restricted by the API gRPC provides. The referenced feature requests want to use specific features in Node's TLS API that gRPC does not expose. The simplest way to address both those requests and any similar future requests is to directly accept any `SecureContext` the user can create.


## Implementation

This is implemented in grpc-js in [PR #1988](https://github.com/grpc/grpc-node/pull/1988)