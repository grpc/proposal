Exposing the checkServerIdentity callback in Node
----
* Author(s): Ian Haken
* Approver: murgatroid99
* Status: Draft
* Implemented in: Node
* Last updated: July 23, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/ucorPSDGIIo

## Abstract

gRPC core now exposes a `verify_peer_callback` option for clients connecting to servers that allows the client to perform additional verification against the peer's presented certificate. This gRFC covers how this callback should be exposed in the Node API.

## Background

The proposal is to base the callback on Node's [tls.checkServerIdentity](https://nodejs.org/api/tls.html#tls_tls_checkserveridentity_hostname_cert) callback. However, the certificate provided to this callback has been parsed into numerous fields whereas the `verify_peer_callback` only provides the raw PEM to the callback. This proposal covers how we should bridge this gap.

It is further worth noting that we want to maintain feature-parity with the grpc-js-core implementation. So the API we choose should be easily interoperable with the callback available in the pure javascript implementation, will would take advantage of Node's built-in checkServerIdentity callback.

### Related Proposals: 
N/A

## Proposal

This proposal suggests that all we expose in the Node gRPC API are the raw DER bytes of the certificate. This would be presented in the `raw` key of an object. To illustrate, this would look like:

```
grpc.credentials.createSsl(
    ca_store,
    client_key,
    client_cert,
    {
      "checkServerIdentity": function(host, cert) {
        /*
        cert = {
          raw: <Buffer ... >
        }
        */
      }
    });
}
```

## Rationale
The lowest common denominator between the fully-parsed certificate made available in Node's callback and the certificate passed in to the callback by grpc-core is the raw certificate (in DER and PEM formats respectively). For this reason, we are suggesting only exposing the raw certificate and leaving it up to consumers of this callback to parse the certificate as desired.

The avoids needing to parse out off the fields of a certificate and trying to match the full format exposed in Node's callback. However, by choosing to place the the raw DER bytes in a Buffer in the `raw` field, this matches Node's behavior with respect to this field and it thus leaves open the option of parsing additional fields to better match Node's implementation in the future.

## Implementation

[Ian Haken](https://github.com/JackOfMostTrades) will be implementing this proposal.

## Open issues (if applicable)

Developers utilizing this new `checkServerIdentity` callback may expect it to behave identically to Node's `checkServerIdentity` callback. I.e. they may expect to be able to apply certificate pinning by asserting `cert.fingerprint === '01:02...'`. The documentation will need to be clear that only the `raw` key is populated in the `cert` parameter of this callback.

