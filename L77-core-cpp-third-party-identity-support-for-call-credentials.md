L77: Core and C++ Third Party Identity Support for Call Credentials
----
* Author(s): Chuan Ren
* Approver: markdroth
* Status: Draft
* Implemented in: core, cpp
* Last updated: 2021-01-27
* Discussion at: https://groups.google.com/g/grpc-io/c/xQn4gSW8LSU

## Abstract

The purpose of this proposal is to enable access to Google APIs directly with 3p identities.

## Background

Currently, gRPC supports 1st party Google token-based authentication. Developers can easily get a `ChannelCredentials` with `grpc::GoogleDefaultCredentials()` and create a rpc channel with it. Notice that the credentials retrieval process happens within `grpc::GoogleDefaultCredentials()`, which is transparent to the users.

### Related Proposals: 
N/A

## Proposal

This doc proposes to extend the access to Google APIS directly with 3p identities. The crednetials of the 3p identities with be in one of the 3 following formats:

**File-source credentials**
```JSON
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/project/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/$PROVIDER_ID",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$EMAIL:generateAccessToken",
  "token_url": "https://sts.googleapis.com/v1beta/token",
  "token_info_url": "https://sts.googleapis.com/v1alpha/token_info",
  "credential_source": {
    "file": "/var/run/secrets/goog.id/token"
  }
}
```

**Url-sourced credentials**
```JSON
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/project/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/$PROVIDER_ID",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$EMAIL:generateAccessToken",
  "token_url": "https://sts.googleapis.com/v1beta/token",
  "token_info_url": "https://sts.googleapis.com/v1alpha/token_info",
  "credential_source": {
    "headers": {
      "Metadata": "True"
    },
    "url": "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://iam.googleapis.com/project/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/$PROVIDER_ID"
  }
}
```

**AWS credentials**
```JSON
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/project/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/$PROVIDER_ID",
  "subject_token_type": "urn:ietf:params:aws:token-type:aws4_request",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$EMAIL:generateAccessToken",
  "token_url": "https://sts.googleapis.com/v1beta/token",
  "token_info_url": "https://sts.googleapis.com/v1alpha/token_info",
  "credential_source": {
    "environment_id": "aws1",
    "region_url": "http://169.254.169.254/latest/meta-data/placement/availability-zone",
    "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials",
    "regional_cred_verification_url": "https://sts.{region}.amazonaws.com?Action=GetCallerIdentity&Version=2011-06-15",
    "cred_verification_url": "https://sts.amazonaws.com?Action=GetCallerIdentity&Version=2011-06-15"
  }
}
```

**Implicit flow**

In C++, `grpc::GoogleDefaultCredentials()` follows the ADC (Application Default Credentials) credentials retrieval process. It will be extended to accept 3p identity credentials. More information on ADC and how they work can be found here: https://developers.google.com/identity/protocols/application-default-credentials

**Explicit flow**

The downside with `GoogleDefaultCredentials()` is that customers will have to use a json credentials file. So besides the implicit flow, a public API will be added in C++ to create an `ExternalAccountCredentials` with explicit 3p identity credentials.
```h
// .../include/grpcpp/security/credentials.h

/// Builds External Account credentials.
/// json_string is the JSON string containing the credentials options.
/// scopes contains the scopes to be binded with the credentials.
std::shared_ptr<CallCredentials>
ExternalAccountCredentials(const grpc::string& json_string, const std::vector<grpc::string>& scopes);
```

To implement that C++ API, the following function will be added to the C-core API:

```h
// .../include/grpc/grpc_security.h

/** Builds External Account credentials.
 - json_string is the JSON string containing the credentials options.
 - scopes_string contains the scopes to be binded with the credentials.
   This API is used for experimental purposes for now and may change in the
 future. */
GRPCAPI grpc_call_credentials* grpc_external_account_credentials_create(
    const char* json_string, const char* scopes_string);
```

## Implementation

1. Base external account credentials class
   https://github.com/grpc/grpc/pull/24208
1. Subclasses to base external account credentials
   1. url-sourced credentials
      https://github.com/grpc/grpc/pull/24411
   1. file-sourced credentials
      https://github.com/grpc/grpc/pull/24526
   1. c-prerequisites. utility to sign Aws requests
      https://github.com/grpc/grpc/pull/24618
   1. aws-sourced credentials
      https://github.com/grpc/grpc/pull/24733
1. Internal and public apis
   https://github.com/grpc/grpc/pull/24814

## Open issues (if applicable)

N/A