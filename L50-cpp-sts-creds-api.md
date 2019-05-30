Title
----
* Author(s): Julien Boeuf
* Approver: a11r
* Status: Draft
* Implemented in: core, cpp
* Last updated: May 20, 2019
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This is a proposal to add support for a new type of credentials in the cpp stack implementing the [OAuth 2.0 Token Exchange](https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16) which documents a protocol for an HTTP- and JSON- based Security Token Service (STS) by defining how to request and obtain security tokens from OAuth 2.0 authorization servers, including security tokens employing impersonation and delegation.

## Background

[OAuth 2.0 Token Exchange](https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16) is the proposed standard for exchanging tokens, for scenarios where a client needs to access a resource that does not natively accept the credentials created by the authentication system used by the client. In these settings, enabling credential exchange can provide better security: Otherwise, the client needs to acquire a different set of credentials to access the resource, and the new credentials must be managed securely.

For instance, Kubernetes workloads can securely obtain credentials from their platform, and they can use these to authenticate to other entities in the cluster, or to the Kubernetes API itself. However, if a Kubernetes workload requires access to a cloud provider's API, it needs a separate set of secrets, and a way to deploy these secrets securely. With credential exchange this is no longer necessary: The workload credential can be directly exchanged for a short-lived token for the cloud API. We expect this type of interaction to become a best practice, as it eliminates a number of problematic patterns, such as secrets checked into code repos, or stored in improperly ACL'ed files.

### Related Proposals:
None.

## Proposal

We are proposing to wrap this token exchange workflow in a new CallCredentials type called StsCredentials (SecureTokenService Credentials):

```cpp
/// Options for creating STS Oauth Token Exchange credentials following the IETF
/// draft https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16.
/// Optional fields may be set to empty string.
struct StsCredentialsOptions {
  grpc::string sts_endpoint_url;     // Required.
  grpc::string resource;             // Optional.
  grpc::string audience;             // Optional.
  grpc::string scope;                // Optional.
  grpc::string requested_token_type; // Optional.
  grpc::string subject_token;        // Required.
  grpc::string subject_token_type;   // Required.
  grpc::string actor_token;          // Optional.
  grpc::string actor_token_type;     // Optional.
};

/// Creates an STS credentials following the STS Token Exchanged specified in
/// the IETF draft
/// https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16.
std::shared_ptr<CallCredentials> StsCredentials(
    const StsCredentialsOptions& options);
```

This API is backed up in core by the following addition in grpc_security.h

```c
/** Options for creating STS Oauth Token Exchange credentials following the IETF
   draft https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16.
   Optional fields may be set to NULL. */
typedef struct {
  const char* sts_endpoint_url;     /* Required. */
  const char* resource;             /* Optional. */
  const char* audience;             /* Optional. */
  const char* scope;                /* Optional. */
  const char* requested_token_type; /* Optional. */
  const char* subject_token;        /* Required. */
  const char* subject_token_type;   /* Required. */
  const char* actor_token;          /* Optional. */
  const char* actor_token_type;     /* Optional. */
} grpc_sts_credentials_options;

/** Creates an STS credentials following the STS Token Exchanged specified in the
   IETF draft https://tools.ietf.org/html/draft-ietf-oauth-token-exchange-16. */
GRPCAPI grpc_call_credentials* grpc_sts_credentials_create(
    const grpc_sts_credentials_options*, void* reserved);
```

## Rationale

A new CallCredentials is the natural way to integrate this functionality in the gRPC framework. The options are per the IETF RFC. Note that we will probably provide a constructor for the options from a JSON file to make integration easier.

Other languages should get STS support from their own authentication library.

## Implementation

1. Adding support in core
    1. grpc_security.h
    1. associated implementation and unit tests
    1. tool (fetch_oauth2) to verify correctness end to end.
1. Adding support in C++

## Open issues (if applicable)
None.
