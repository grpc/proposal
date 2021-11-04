Remove grpc_insecure_channel_create and grpc_server_add_insecure_http2_port from Core Surface API
----
* Author(s): yihuazhang
* Approver: markdroth
* Status: {In Review}
* Implemented in: https://github.com/grpc/grpc/pull/25586
* Last updated: 2021-11-01
* Discussion at: https://groups.google.com/g/grpc-io/c/9B4-AHCx_ZM 

## Abstract

* Remove `grpc_insecure_channel_create` and `grpc_server_add_insecure_http2_port` from Core Surface API.
* Rename `grpc_server_add_secure_http2_port` to `grpc_server_add_http2_port`.
* Rename `grpc_secure_channel_create` to `grpc_channel_create` and re-order its parameters.
* Move `grpc_channel_credentials` and `grpc_server_credentials`related APIs from `grpc_security.h` to `grpc.h`.
* Add `grpc_channel_create_from_fd` and `grpc_server_add_channel_from_fd` to Core Surface API.

## Background

Currently, the gRPC insecure build does not include any of the functions in `grpc_security.h`, nor does it include any of the code in `src/core/lib/security`. This means that it does not support channel or call credentials, security connectors, or client/server auth filters. It also does not link in SSL libraries. We will stop supporting insecure builds, so that all builds of gRPC include the functions in `grpc_security.h` and the code in `src/core/lib/security`. There are several reasons for getting rid of insecure builds:

* The current approach requires different API entry points for secure vs. insecure channels and servers, and it requires a lot of conditional compilation that complicates our build system.
* The main reason for supporting insecure builds is because there are some users that don't need security but really need to minimize binary size (e.g., embedded systems), so they need a way to build without depending on the SSL libraries. We will still satisfy this requirement after getting rid of insecure builds.
* The current approach was decided upon early in our development, before we had the credential APIs, and it was never updated when the credential APIs were added. However, the credential APIs provide a much cleaner way to allow building gRPC without the SSL dependency.

To solve this problem, we are going to move to an approach where insecure
channels and servers are handled the same way as secure channels and servers. It
will no longer be possible to build gRPC without support for credentials or to
create a channel or server without specifying credentials, but we will provide
an insecure credentials type to be used in cases where security is not needed.
The insecure credentials type will not depend on SSL libraries and will be
included in all gRPC builds. Other credential types that do depend on SSL
libraries will be available in secure builds only, just as they are today.
Also, the gRPC build will not directly include the SSL libraries.
Instead, there will be separate build targets for each of the credential types, and
only those credential types that need SSL will pull in the SSL library dependencies.


## Proposal

First, move `grpc_channel_credentials` and `grpc_server_credentials`related APIs from `grpc_security.h` to `grpc.h`.

```
/** --- grpc_channel_credentials object. ---
   A channel credentials object represents a way to authenticate a client on a
   channel. Different types of channel credentials are declared in
   grpc_security.h. */

typedef struct grpc_channel_credentials grpc_channel_credentials;

/** Releases a channel credentials object.
   The creator of the credentials object is responsible for its release. */

GRPCAPI void grpc_channel_credentials_release(grpc_channel_credentials* creds);

/** --- grpc_server_credentials object. ---
   A server credentials object represents a way to authenticate a server.
   Different types of server credentials are declared in grpc_security.h. */

typedef struct grpc_server_credentials grpc_server_credentials;

/** Releases a server_credentials object.
   The creator of the server_credentials object is responsible for its release.
   */
GRPCAPI void grpc_server_credentials_release(grpc_server_credentials* creds);

```

Second, rename `grpc_secure_channel_create` to `grpc_channel_create` and re-order its parameters. Then, move it from `grpc_security.h` to `grpc.h`.


```
/** Creates a secure channel using the passed-in credentials. Additional
    channel level configuration MAY be provided by grpc_channel_args, though
    the expectation is that most clients will want to simply pass NULL. The
    user data in 'args' need only live through the invocation of this function.
    However, if any args of the 'pointer' type are passed, then the referenced
    vtable must be maintained by the caller until grpc_channel_destroy
    terminates. See grpc_channel_args definition for more on this. */
GRPCAPI grpc_channel* grpc_channel_create(const char* target,
                                          grpc_channel_credentials* creds,
                                          const grpc_channel_args* args);
```

Third, rename `grpc_server_add_secure_http2_port` to `grpc_server_add_http2_port` and move it from `grpc_security.h` to `grpc.h`.

```
/** Add a HTTP2 over an encrypted link over tcp listener.
   Returns bound port number on success, 0 on failure.
   REQUIRES: server not started */
GRPCAPI int grpc_server_add_http2_port(grpc_server* server, const char* addr,
                                       grpc_server_credentials* creds);
```

Fourth, remove `grpc_insecure_channel_create_from_fd` and `grpc_server_add_insecure_channel_from_fd`,
and add `grpc_channel_create_from_fd` and `grpc_server_add_channel_from_fd` instead. Note that the new APIs only support
either insecure channel credentials (at client-side) or insecure server credentials (at server-side), and using other types of
credentials will result in a failure. In future, the APIs will be improved to support other types of credentials.


```
/** Create a secure channel to 'target' using file descriptor 'fd' and passed-in
    credentials. The 'target' argument will be used to indicate the name for
    this channel. Note that this API currently only supports insecure channel
    credentials. Using other types of credentials will result in a failure. */
GRPCAPI grpc_channel* grpc_channel_create_from_fd(
    const char* target, int fd, grpc_channel_credentials* creds,
    const grpc_channel_args* args);

/** Add the connected secure communication channel based on file descriptor 'fd'
   to the 'server' and server credentials 'creds'. The 'fd' must be an open file
   descriptor corresponding to a connected socket. Events from the file
   descriptor may come on any of the server completion queues (i.e completion
   queues registered via the grpc_server_register_completion_queue API).
   Note that this API currently only supports inseure server credentials
   Using other types of credentials will result in a failure.
   TODO(hork): add channel_args to this API to allow endpoints and transports
   created in this function to participate in the resource quota feature. */
GRPCAPI void grpc_server_add_channel_from_fd(grpc_server* server,
                                             int fd,
                                             grpc_server_credentials* creds);

```
Fifth, remove `grpc_insecure_channel_create` and `grpc_server_add_insecure_http2_port` from `grpc.h`. After this change takes effect, the only way of creating gRPC insecure channels will be first creating insecure channel or server credentials, and then invoking `grpc_channel_create` or `grpc_server_add_http2_port` at client or server side, respectively.

## Rationale

* By removing gRPC insecure build, gRPC build system will be simplified. We do not need to maintain two implementations of certain features - one for secure build and another for insecure one. `grpclb_channel.cc` and `grpclb_channel_secure.cc` is an example.

* From security perspective, if there is a need to restrict the usage of gRPC insecure channels, the problem is simplified to some extent because we only need to control a single means of creating insecure channels, instead of two.

* Most importantly, since we introduce a different target for each credential type, for the binaries that do not rely on SSL-related functionalities, they do not need to bring in SSL dependency and thus will have a smaller binary size.

* The proposed change only affects users who directly interact with Core surface APIs (e.g., by calling insecure channel creation APIs). For the users who interact with wrapped language implementations, the change will be no-op.

## Implementation

Core: https://github.com/grpc/grpc/pull/25586/files

## Open issues (if applicable)

N/A
