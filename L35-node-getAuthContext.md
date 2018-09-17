Exposing the per-call authentication context data in Node
----
* Author(s): Nicolas Noble
* Approver: murgatroid99
* Status: Draft
* Implemented in: Node
* Last updated: August 21, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/yVnvHDGxTME

## Abstract

gRPC-node now exposes a `getAuthContext` method on the call objects that allows clients and servers to retreive the authentication context information. This gRFC covers how this new method should be exposed in the Node API.

## Background

The proposal is to get the new `getAuthContext` method to return a Node's idiomatic object, hiding away the iterative structure of the core's auth_context.

We want to maintain feature-parity with the pure javascript implementation of gRPC. While the spirit of the core's auth_context is to be opaque and extensive, and let it up to the developers to interpret the properties, we need to be able to emulate it in the pure javascript implementation, and guarantee feature parity. This means we need to whitelist fields we are reading from the native core, and transform them in a way that we know can work similarly between the two implementations.

It is worth noting that a sort of equivalent API call in the Node's runtime is [tls.TLSSocket.getPeerCertificate](https://nodejs.org/api/tls.html#tls_tlssocket_getpeercertificate_detailed), which returns an actual SSL certificate. The only common field in this object we can guarantee to be identical between the two implementations is the `raw` line, that contains a Buffer with the binary representation of the peer certificate.

Therefore, this proposal offers to only cover two fields at the beginning:
 - `transport_security_type`, transformed into a singleton string `transportSecurityType`
 - `x509_pem_cert`, transformed into the object: `sslPeerCertificate: { raw: certificateBuffer }`

### Related Proposals: 
N/A

## Proposal

This proposal suggests that all we expose in the Node gRPC API are the raw DER bytes of the certificate for the `x509_pem_cert` field if present, and the singleton element `transport_security_type`. This would be presented in the `raw` key of an object. To illustrate, this would look like:

```
const authContext = call.getAuthContext()
/*
authContext = {
  transportSecurityType: 'ssl',
  sslPeerCertificate: {
    raw: <Buffer ... >
  }
}
*/
```

## Rationale
The lowest common denominator between the fully-parsed certificate made available in Node's getPeerCertificate method and the certificate stored in the auth_context by grpc-core is the raw certificate (in DER and PEM formats respectively). For this reason, we are suggesting only exposing the raw certificate and leaving it up to consumers of this callback to parse the certificate as desired.

The avoids needing to parse out off the fields of a certificate and trying to match the full format exposed in Node's getPeerCertificate method. However, by choosing to place the the raw DER bytes in a Buffer in the `raw` field, this matches Node's behavior with respect to this field and it thus leaves open the option of parsing additional fields to better match Node's implementation in the future.

## Implementation

[Nicolas Noble](https://github.com/nicolasnoble) will be implementing this proposal.

## Open issues (if applicable)

Developers utilizing this new `getAuthContext` method may expect it to behave similar to Node's `getPeerCertificate` method. I.e. they may expect to be able to apply certificate pinning by asserting `cert.fingerprint === '01:02...'`. The documentation will need to be clear that only the `raw` key is populated in the `sslPeerCertificate` property.
