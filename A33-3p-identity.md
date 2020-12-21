3p Identity Supoort
----
* Author(s): Chuan Ren
* Approver: a11r
* Status: Draft
* Implemented in: core, cpp
* Last updated: 2020-12-21
* Discussion at: 

## Abstract

Enable access to Google APIs directly with 3p identities.

## Background

Currently, gRPC supports 1st party Google token-based authentication. Developers can easily get a `ChannelCredentials` with `grpc::GoogleDefaultCredentials()` and create a rpc channel with it. Notice that the credentials retrieval process happens within `grpc::GoogleDefaultCredentials()`, which is transparent to the users.

### Related Proposals: 
N/A

## Proposal

This doc propases to extend the access to Google APIS directly with 3p identities. The crednetials of the 3p identities with be in one of the 3 following formats:

File-source credentials
```
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

Url-sourced credentials
```
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

AWS credentials
```
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/project/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/"$PROVIDER_ID",
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

Implicit flow
In core, the underlying credentials retrieval process of `grpc::GoogleDefaultCredentials()` will be extended to accept 3p identity credentials implicitly.

Explicit flow
In core and cpp, a public api will be added to create an `ExternalAccountCredentials` with explicit 3p identity credentials.
```
// .../include/grpcpp/security/credentials.h

std::shared_ptr<CallCredentials>
ExternalAccountCredentials(const grpc::string& json_string, const std::vector<grpc::string>& scopes);
```

## Implementation

1. Base external account credentials class
   https://github.com/grpc/grpc/pull/24208
2. Subclasses to base external account credentials
   a. url-sourced credentials
      https://github.com/grpc/grpc/pull/24411
   b. file-sourced credentials
      https://github.com/grpc/grpc/pull/24526
   c-prerequisites. utility to sign Aws requests
      https://github.com/grpc/grpc/pull/24618
   c. aws-sourced credentials
      https://github.com/grpc/grpc/pull/24733
3. Internal and public apis
   https://github.com/grpc/grpc/pull/24814

## Open issues (if applicable)

N/A