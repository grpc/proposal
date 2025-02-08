Exposing the Peer Certificate in gRPC-JS Server Call Context

----
* Author(s): [David Fiala](https://github.com/davidfiala), Nicolas Noble
* Approver: murgatroid99
* Status: Draft
* Implemented in: Node.js
* Last updated: May 28, 2024
* Discussion at: https://github.com/grpc/grpc-node/issues/2730, https://groups.google.com/forum/#!topic/grpc-io/yVnvHDGxTME

## Abstract

This proposal exposes the peer certificate in gRPC-JS, allowing servers to access the client's TLS certificate as part of the call context. This feature is essential for mutual authentication and secure communications involving TLS authn, key pinning, and more.

## Background

The gRPC-JS library currently lacks a mechanism for servers to access the client's TLS certificate, a feature critical for mutual authentication. This proposal addresses this gap by introducing a method to expose the peer certificate in a manner consistent with modern Node.js and TypeScript practices.

### Related Proposals: 
N/A

### History

In 2018, an initial proposal was made to expose the certificate's bytes in gRPC-Node and gRPC-JS. However, this was never implemented. With the deprecation of native node-grpc, we now explore options that are more aligned with Node.js and JavaScript best practices, ensuring a modern, efficient, and idiomatic solution.

## Proposal

Introduce a method in the gRPC-JS library to retrieve the client's peer certificate. The proposed method will return the certificate information as provided by Node.js's `tls.TLSSocket.getPeerCertificate` method. To aid present and future compatibility, for secure connections we will pass any arguments and return value to and from `tls.TLSSocket.getPeerCertificate` directly and match node's semantics. For non-secure gRPC connections, we will return an empty object.

### API Design

The new method, `getPeerCertificate`, will be added to the gRPC server call object. This method will return the output of `tls.TLSSocket.getPeerCertificate`, either as a detailed certificate or a simplified version, based on the specified parameter. For non-secure channels, we will return an empty object keeping in line with the spirit of node's `getPeerCertificate` which states, `If the peer does not provide a certificate, an empty object will be returned`.

### Example Usage

```typescript
import { Server, ServerUnaryCall, sendUnaryData } from '@grpc/grpc-js';

function myServiceMethod(call: ServerUnaryCall<any, any>, callback: sendUnaryData<any>) {
  const peerCert = call.getPeerCertificate();
  console.log('Peer Certificate:', peerCert);
  // Implement additional logic using the peer certificate
  callback(null, { message: 'Success' });
}

const server = new Server();
server.addService(myServiceDefinition, { myServiceMethod });
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createSsl(null, [{ private_key: privateKey, cert_chain: certChain }]), () => {
  server.start();
});
```

### Rationale

Exposing the peer certificate is essential for mutual TLS, where the certificate contains the peer's identity among other information. This practice is used internally by Google with LOAS and is necessary for robust security implementations. We want such practices to be possible externally and even encouraged as part of robust mutual TLS authentication.

### Performance Considerations

By providing access to the full certificate, we ensure that developers do not have to deserialize and parse it multiple times, enhancing performance and efficiency.

### Implementation

[David Fiala](https://github.com/davidfiala) will provide the implementation.

The implementation will involve modifying the gRPC-JS server call object to include the `getPeerCertificate` method, with parameters being inferred from grpc-js. On secure connections, it will directly call and return `tls.TLSSocket.getPeerCertificate` including any arguments passed by the user. For the return value, inferred types from node's TLS types are used to ensure that future API and/or interface changes in node are reflected transparently in grpc-js. On non-secure connections, we will short-circuit and return an empty object.
