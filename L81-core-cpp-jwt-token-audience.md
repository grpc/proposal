L81: Custom Audience in JWT Access Credentials and Google Default Credentials
----
* Author(s): [Jiangtao Li](https://github.com/jiangtaoli2016), [Yihua Zhang](https://github.com/yihuazhang)
* Approver: markdroth
* Status: Implemented {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: Core, C++
* Last updated: 2021-06-04
* Discussion at: https://groups.google.com/g/grpc-io/c/MgSP5tNrM0Q

## Abstract

In current gRPC C++ implementation, the audience field of JWT access token is
always set to the service URL. It works for most of the cases. However, there
are use cases where a user wants to override the default audience (service URL)
and uses its own. This gRFC provides a way for an applications to override
the default audience field.

## Background

Consider an application wants to use JWT Access Credentials or Google Default
Credentials to connect to a regional endpoint, e.g., 
`us-dialogflow.googleapis.com`, rather than the global endpoint 
`dialogflow.googleapis.com`, the cloud service may reject the JWT because the
auidence field was set to `us-dialogflow.googleapis.com`. In such scenarios,
the application wants the ability to override the default audience field and
set the auidence field of its choice.

### Related Proposals: 

NA

## Proposal

We propose to change the following C++ API. If the user_provided_audience
field is empty, then service URL will be used as the auidence field, otherwise,
the user_provided_audience will be used as the audience in the JWT.

When the user_provided_audience field is set in Google Default Credentials,
this field is only used when a service account JWT access credential is
created. This field will be no-op if for example a compute engine credential
is created.

```
/// Builds Service Account JWT Access credentials.
/// json_key is the JSON key string containing the client's private key.
/// token_lifetime_seconds is the lifetime in seconds of each Json Web Token
/// (JWT) created with this credentials. It should not exceed
/// \a kMaxAuthTokenLifetimeSecs or will be cropped to this value.
/// user_provided_audience is an optional field for user to override the
/// auidence in the JWT token. If user_provided_audience is empty, the service
/// URL will be used as the audience.
std::shared_ptr<CallCredentials> ServiceAccountJWTAccessCredentials(
    const grpc::string& json_key,
    long token_lifetime_seconds = kMaxAuthTokenLifetimeSecs,
    const grpc::string& user_provided_audience = "");


/// Builds credentials with reasonable defaults.
/// user_provided_audience is an optional field for user to override the
/// auidence in the JWT token. If user_provided_audience is empty, the service
/// URL will be used as the audience. Note that user_provided_audience will
/// only be used if a service account JWT access credential is created by
/// the application default credentials mechanism.
std::shared_ptr<ChannelCredentials> GoogleDefaultCredentials(
    const grpc::string& user_provided_audience = "");
```

We also propose to change the C core API as follows

```
/** Creates a JWT credentials object. May return NULL if the input is invalid.
   - json_key is the JSON key string containing the client's private key.
   - token_lifetime is the lifetime of each Json Web Token (JWT) created with
     this credentials.  It should not exceed grpc_max_auth_token_lifetime or
     will be cropped to this value.
   - user_provided_audience is an optional field for user to override the
     auidence in the JWT token. If user_provided_audience is empty, the
     service URL will be used as the audience.  */
GRPCAPI grpc_call_credentials*
grpc_service_account_jwt_access_credentials_create(
    const char* json_key, gpr_timespec token_lifetime,
    const char* user_provided_audience);

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

   user_provided_audience is an optional field for user to override the
   audience in the JWT token if used. If user_provided_audience is empty,
   the service URL will be used as the audience. Note that 
   user_provided_audience will only be used if a service account JWT access
   credential is created by the application default credentials mechanism. Also
   note that user_provided_audience will be ignored if the call_credentials is
   not nullptr.
*/
GRPCAPI grpc_channel_credentials* grpc_google_default_credentials_create(
    grpc_call_credentials* call_credentials,
    const char* user_provided_audience);
```

### Temporary environment variable protection

The proposed change in this gRFC is backward compatible since it is the default
value of user_provided_audience is an empty string, thus the default service
URL is used as the auidence the JWT.

## Rationale

NA

## Implementation

Implementation is straight-forward. We just need to plumb the
user_provided_audience to [jwt_credentials.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/security/credentials/jwt/jwt_credentials.cc), 
so that this field can be set as the audience field, if it is not an empty
string.

## Open issues (if applicable)

NA
