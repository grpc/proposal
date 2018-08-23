Enhance createSsl's arguments
----
* Author(s): Nicolas Noble
* Approver: murgatroid99
* Status: Draft
* Implemented in: Node
* Last updated: August 22, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/YwCF1bKgiXc

## Abstract
The arguments to gRPC-node's `createSsl` have gotten in a state with too much optional parameters, where a plain old javascript object would be more desirable.

## Background
As the API of gRPC-node evolved, the `createSsl` method gained multiple optional arguments. This has gotten us in a situation where this API is now in an anti-pattern where users need to count the number of `null`s to skip the arguments they don't care about until they finally arrive to the ones that matters to them.

Additionally, the exposed API has fallen behind with the changes from the core. In particular:

 - The boolean argument `checkClientCertificate` has become an enum, with the following values:
   - `GRPC_SSL_DONT_REQUEST_CLIENT_CERTIFICATE`
   - `GRPC_SSL_REQUEST_CLIENT_CERTIFICATE_BUT_DONT_VERIFY`
   - `GRPC_SSL_REQUEST_CLIENT_CERTIFICATE_AND_VERIFY`
   - `GRPC_SSL_REQUEST_AND_REQUIRE_CLIENT_CERTIFICATE_BUT_DONT_VERIFY`
   - `GRPC_SSL_REQUEST_AND_REQUIRE_CLIENT_CERTIFICATE_AND_VERIFY`
 - There is new optional callback for the server credentials, to emit the credential information for every new connection.

### Related Proposals:
N/A

## Proposal
The first argument to `createSsl` is always the optional CA certificate, so it can be `null`, `undefined`, or a `Buffer`. This proposal adds a new behavior: if createSsl has exactly one argument, and if this argument is a plain object, then use the object's data as the arguments to `createSsl`. This will also allows for a more flexible way to add new parameters in the future.

First, we propose to add a few more constants:
```
enum ssl.clientCertificateRequest {
  DONT_REQUEST_CLIENT_CERTIFICATE = 0,
  REQUEST_CLIENT_CERTIFICATE_BUT_DONT_VERIFY = 1,
  REQUEST_CLIENT_CERTIFICATE_AND_VERIFY = 2,
  REQUEST_AND_REQUIRE_CLIENT_CERTIFICATE_BUT_DONT_VERIFY = 3,
  REQUEST_AND_REQUIRE_CLIENT_CERTIFICATE_AND_VERIFY = 4,
}
```

Then there are two variants of `createSsl`: the one from `grpc.credentials`, and the one from `grpc.ServerCredentials`. This proposal offers two definitions for the new objects that can be used as sole argument to these functions:
```
type CertificateConfigType = {
  ca?: Buffer,
  keyCertPairs?: KeyCertPair | KeyCertPair[],
}
type ServerSslOptions = {
  certificateConfig?: CertificateConfigType | function(): null | CertificateConfigType | Error,
  checkClientCertificate?: ssl.clientCertificateRequest
}
type ClientSslOptions = {
  ca?: Buffer,
  keyCertPair?: KeyCertPair,
  checkServerIdentity?: CheckServerIdentityCallback,
}
```

## Rationale
Changing the API completely would be a breaking change. Another method would be to create new function names, suffixed with `ex` for instance. But this proposal with its flexibility is typical of other NodeJS' APIs way of dealing with API evolutions like these.

Also, we want to make sure any new API introduced is compatible with the pure javascript implementation. All of the proposed changes are implementable in pure javascript, using the Node's TLS API.

## Implementation
[Nicolas Noble](https://github.com/nicolasnoble) will be implementing this proposal.

## Open issues (if applicable)
N/A

