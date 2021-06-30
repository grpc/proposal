L81: Custom Audience in JWT Access Credentials and Google Default Credentials
----
* Author(s): [Jiangtao Li](https://github.com/jiangtaoli2016), [Yihua Zhang](https://github.com/yihuazhang)
* Approver: markdroth
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: Core, C++
* Last updated: 2021-06-29
* Discussion at: https://groups.google.com/g/grpc-io/c/MgSP5tNrM0Q

## Abstract

In current gRPC C++ implementation, it does not allow to include the scope
field in a JWT access token which makes it impossible for the token
to be used in a situation where a token consumer explicitly looks for the scope
field. This gRFC provides a way for an application to include a user-provided scope
in a JWT access token.

## Background

Consider an application wants to use JWT Access Credentials or Google Default
Credentials to connect to a regional endpoint, e.g.,
`us-dialogflow.googleapis.com`. It requires to either explicitly set the scope -
`https://www.googleapis.com/auth/cloud-platform` or override the default
audience which is currenlty set to the service URL to `us-dialogflow.googleapis.com`.
The current gRPC C++
implementation does not support either option. However since it is recommended by
the Google API (https://google.aip.dev/auth/4111) to use scope
instead of audience, we decide to extend the existing implementation to support
the inclusion of scope in a JWT access token. It also makes the behavior
consistent across other gRPC language stacks.

### Related Proposals: 

NA

## Proposal

We propose to change the following C++ API. If the user_provided_scope
field is not empty, we will include it as the scope field in the JWT.
However if clear_audience is set to true, we additionally clear
the audience field in the JWT, and the generated token will be used to
access the Google APIs that explicitly requires audience and scope fields
cannot co-exist (https://google.aip.dev/auth/4111).

When the user_provided_scope field is set in Google Default Credentials,
this field is only used when a service account JWT access credential is
created. This field will be no-op if for example a compute engine credential
is created.

```
/// Builds Service Account JWT Access credentials.
/// json_key is the JSON key string containing the client's private key.
/// token_lifetime_seconds is the lifetime in seconds of each Json Web Token
/// (JWT) created with this credentials. It should not exceed
/// \a kMaxAuthTokenLifetimeSecs or will be cropped to this value.
/// user_provided_scope is an optional field for user to use to represent the
/// scope field in the JWT token.
/// clear_audience is a boolean field that dictates clearing audience
/// field in the JWT token when it is set to true AND user_provided_scope is
/// not empty. */
std::shared_ptr<CallCredentials> ServiceAccountJWTAccessCredentials(
    const grpc::string& json_key,
    long token_lifetime_seconds = kMaxAuthTokenLifetimeSecs,
    const grpc::string& user_provided_scope = "",
    bool clear_audience);


/// Builds credentials with reasonable defaults.
/// user_provided_scope is an optional field for user to use to represent the
/// scope field in the JWT token. It will only be used if a service account JWT
/// access credential is created by the application default credentials
/// mechanism. If user_provided_scope is not empty, the audience (service URL)
/// field in the JWT token will be cleared, which is dictated by
/// https://google.aip.dev/auth/4111.
std::shared_ptr<ChannelCredentials> GoogleDefaultCredentials(
    const grpc::string& user_provided_scope = "");
```

We also propose to change the C core API as follows

```
/** Creates a JWT credentials object. May return NULL if the input is invalid.
   - json_key is the JSON key string containing the client's private key.
   - token_lifetime is the lifetime of each Json Web Token (JWT) created with
     this credentials.  It should not exceed grpc_max_auth_token_lifetime or
     will be cropped to this value.
   - user_provided_scope is an optional field for user to use in the JWT token
     to represent the scope field.
   - clear_audience dictating the clearance of audience field when it
     is set to a non-zero value AND user_provided_scope is not nullptr. */
GRPCAPI grpc_call_credentials*
grpc_service_account_jwt_access_credentials_create(
    const char* json_key, gpr_timespec token_lifetime,
    const char* user_provided_scope,
    int clear_audience);

/** Creates default credentials to connect to a google gRPC service.
   WARNING: Do NOT use this credentials to connect to a non-google service as
   this could result in an oauth2 token leak. The security level of the
   resulting connection is GRPC_PRIVACY_AND_INTEGRITY.

   If specified, the supplied call credentials object will be attached to the
   returned channel credentials object. The call_credentials object must remain
   valid throughout the lifetime of the returned grpc_channel_credentials
   object. It is expected that the call credentials object was generated
   according to the Application Default Credentials mechanism and asserts the
   identity of the default service account of the machine. Supplying any other
   sort of call credential will result in undefined behavior, up to and
   including the sudden and unexpected failure of RPCs.

   If nullptr is supplied, the returned channel credentials object will use a
   call credentials object based on the Application Default Credentials
   mechanism.

   user_provided_scope is an optional field for user to use in the JWT token to
   represent the scope field. It will only be used if a service account JWT
   access credential is created by the application default credentials
   mechanism. If user_provided_scope is not nullptr, the audience (service URL)
   field in the JWT token will be cleared, which is dictated by
   https://google.aip.dev/auth/4111. Also note that user_provided_scope will be
   ignored if the call_credentials is not nullptr.
*/
GRPCAPI grpc_channel_credentials* grpc_google_default_credentials_create(
    grpc_call_credentials* call_credentials,
    const char* user_provided_audience);
```

### Temporary environment variable protection

The proposed change in this gRFC is backward compatible since the default
value of user_provided_scope is an empty string and the default value of 
clear_audience is false. Thus, it does not change the existing behavior
that uses the service URL as the auidence in the JWT.

## Rationale

NA

## Implementation

Implementation is straight-forward. We just need to plumb the
user_provided_scope and clear_audience to [jwt_credentials.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/security/credentials/jwt/jwt_credentials.cc), 
so that user_provided_scope can be set as the scope field, if it is not an empty
string.

## Open issues (if applicable)

NA
