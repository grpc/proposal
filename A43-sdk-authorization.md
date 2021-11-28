A43: SDK authorization
----
* Author(s): [Ashitha Santhosh](https://github.com/ashithasantosh)
* Approver: [Doug Fawley](https://github.com/dfawley),
            [Eric Anderson](https://github.com/ejona86),
            [Mark Roth](https://github.com/markdroth),
            [Sanjay Pujare](https://github.com/sanjaypujare),
            [Yihua Zhang](https://github.com/yihuazhang)
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-08-30
* Discussion at: https://groups.google.com/g/grpc-io/c/m4Lns0tO-fE

## Abstract

gRPC is currently implementing an open-source standard authorization solution.
This proposal discusses how this framework would be made available to OSS gRPC
users (like network switches) to perform per-RPC authorization checks.

## Background

Authentication verifies the identity of the requestor. Authorization ensures that
the requestor has sufficient access to make a particular request.

gRPC offers built-in authentication mechanisms like TLS, ALTS and even allow
users to plug in their own authentication systems. However, gRPC doesn't have any
standard authorization solution available. If a user wanted to perform per RPC
authorization check, they have to implement their own solution in the application.
The standard gRPC authorization mechanism described in this proposal allows users
to simply provide an authorization policy for their server. Then, for each RPC,
the server verifies whether the client is authorized to make the RPC request.

## Proposal

### User facing Authorization Policy

gRPC SDK authorization policy is the user facing policy language that allows
service owners and/or security admins to enable per-RPC authorization checks.

SDK authorization policy is extendable, we may add new fields in the future as
new requirements are introduced, we won’t be removing fields or updating
semantics of existing fields. However, this means if user wants to use policy
with new fields, they need to use the right gRPC version. Using policy with new
fields, with an older version of gRPC, will result in policy being marked invalid.
This is essential, because otherwise we may end up allowing an RPC request without
even evaluating the new field values in policy, which poses a security risk.

Following is the JSON schema of SDK Authorization Policy Version 1.0

```json
{
  "title": "AuthorizationPolicy",
  "definitions": {
  	"rule": {
  		"description": "Specification of rules. An empty rule is always matched"
  		  "(i.e., both source and request are empty)",
  		"type" : "object",
  		"properties": {
  			"name": {
  				"description": "The name of an authorization rule. This name should"
  				  "be unique within the list of deny (or allow) rules. It is mainly"
  				  "for monitoring and error message generation.",
  				"type": "string"
  			},
  			"source": {
  				"description": "Specifies attributes of a peer. Fields in the source"
  				  "are ANDed together, once we support multiple fields in the future."
  				  "If not set, no checks will be performed against the source.",
  				"type": "object",
  				"properties": {
  					"principals": {
  						"description": "A list of peer identities to match for"
  						  "authorization. The principals are one of, i.e., it matches"
  						  "if one of the principals matches."
  						  "The field supports Exact, Prefix, Suffix and Presence matches."
  						  "- Exact match: \"abc\" will match on value \"abc\"."
  						  "- Prefix match: \"abc*\" will match on value \"abc\" and \"abcd\"."
  						  "- Suffix match: \"*abc\" will match on value \"abc\" and \"xabc\"."
  						  "- Presence match: \"*\" will match when the value is not empty.",
  						"type": "array",
  						"items": {
  							"type": "string"
  						}
  					}
  				}
  			},
  			"request": {
  				"description": "Specifies attributes of a request. Fields in the"
  				  "request are ANDed together. If not set, no checks will be performed"
  				  "against the request.",
  				"type": "object",
  				"properties": {
  					"paths": {
  						"description": "A list of paths to match for authorization. This is"
  						  "the fully qualified name in the form of \"/package.service/method\"."
  						  "The paths are ORed together, i.e., it matches if one of the"
  						  "paths matches."
  						  "This field supports Exact, Prefix, Suffix and Presence matches."
  						  "- Exact match: \"abc\" will match on value \"abc\"."
  						  "- Prefix match: \"abc*\" will match on value \"abc\" and \"abcd\"."
  						  "- Suffix match: \"*abc\" will match on value \"abc\" and \"xabc\"."
  						  "- Presence match: \"*\" will match when the value is not empty.",
  						"type": "array",
  						"items": {
  							"type": "string"
  						}
  					},
  					"headers": {
  						"description": "A list of HTTP header key/value pairs to match"
  						  "against, for potentially advanced use cases. The headers are"
  						  "ANDed together, i.e., it matches only if *all* the headers"
  						  "match.",
  						"type": "array",
  						"items": {
  							"type": "object",
  							"properties": {
  								"key": {
  									"description": "The name of the HTTP header to"
  									  "match. The following headers are *not*"
  									  "supported: the \"Host\" header, \"hop-by-hop\""
  									  "headers (e.g. those listed in \"Connection\""
  									  "header), HTTP/2 pseudo headers (\":\"-prefixed)"
  									  "and headers prefixed with \"grpc-\".",
  									"type": "string"
  								},
  								"values": {
  									"description": "A list of header values to"
  									  "match. The header values are ORed together,"
  									  "i.e., it matches if one of the values"
  									  "matches. Multi-valued headers are considered"
  									  "a single value with commas added between"
  									  "values."
  									  "This field supports Exact, Prefix, Suffix"
  									  "and Presence match."
  									  "- Exact match: \"abc\" will match on value"
  									  "  \"abc\"."
  									  "- Prefix match: \"abc*\" will match on value"
  									  "  \"abc\" and \"abcd\"."
  									  "- Suffix match: \"*abc\" will match on value"
  									  "  \"abc\" and \"xabc\"."
  									  "- Presence match: \"*\" will match when the"
  									  "  value is not empty.",
  									"type": "array",
  									"items": {
  										"type": "string"
  									}
  								}
  							},
  							"required": ["key", "values"]
  						}
  					}
  				}
  			}
  		},
  		"required": ["name"]
  	}
  },
  "description": "AuthorizationPolicy defines which principals are permitted to"
    "access which resource. Resources are RPC methods scoped by services.",
  "type": "object",
  "properties": {
  	"name": {
  		"description": "The name of an authorization policy. It is mainly for"
  		  "monitoring and error message generation.",
  		"type": "string"
  	},
  	"deny_rules": {
  		"description": "List of deny rules to match. If a request matches any of the"
  		  "deny rules, then it will be denied. If none of the deny rules matches or"
  		  "there are no deny rules, the allow rules will be evaluated.",
  		"type": "array",
  		"items": {
  			"$ref": "#/definitions/rule"
  		}
  	},
  	"allow_rules": {
  		"description": "List of allow rules to match. The allow rules will only be"
  		  "evaluated after the deny rules. If a request matches any of the allow"
  		  "rules, then it will allowed. If none of the allow rules matches, it will"
  		  "be denied.",
  		"type": "array",
  		"items": {
  			"$ref": "#/definitions/rule"
  		}
  	}
  },
  "required": ["name", "allow_rules"]
}
```

#### Details

gRPC SDK authorization policy has a list of *allow rules* and a list of *deny rules*.
The following sequence is followed to make an authorization decision -
1. Check for a match in *deny rules*, if a match is found, the request will be denied.
   If no match is found in deny rules, or there are no deny rules execute next step.
2. Check for a match in *allow rules*, if a match is found, the request will be allowed.
   Note that allow rules is a required field here, if not present, policy is invalid.
3. If no match is found in *allow rules*, we deny the request.

Each *rule* has the following semantics -
1. Each *rule* has a *name*, a *source* and a *request* to match against. For a *rule*
   to match, both *source* and *request* must match.
   - If both *source* and *request* are empty, the *rule* always matches. It is a
     wildcard that can be used in a deny list to deny all RPCs or in an allow list to
     accept all RPCs except those matching in the deny list.
   - If only *source* is empty, we evaluate against the *request* fields and apply
     that to any user (See example below). Similarly, if only *request* is empty, we
     evaluate against the *source* fields and apply that to any action.
2. Each *source* could contain a list of *principals*. The *principals* are ORed
   together, i.e. it matches if one of them matches.
   Sequence of steps to evaluate each principal from config -
   1. If TLS is not used, matching fails.
   2. If there is no client certificate, we get a match if principal is an empty
      string.
   3. If we have a client certificate, we check against certificate contents. A
      match is found if principal matches URI SANs from the certificate. If
      there are no URI SANs in certificate, or no match was found, we check
      against DNS SANs. Similarly, if certificate has no DNS SANs or match wasn't
      found, we check against Subject field from certificate.
   
   Consider the case where the *principals* list is empty, then we only check for
   step 1 above, i.e. rule matches for any authenticated user.
3. Each *request* could contain a list of URL *paths* (i.e. fully qualified RPC
   methods) and list of http *headers* to match. Refer JSON schema above to
   understand matching semantics.
   - If the sub-fields are empty, like if the *paths* is empty (assuming *headers*
     is unset), the behavior would be similar to an empty *request* since no other
     *request* fields are set.

#### Example

In the following policy example
- Peer identity from ["spiffe://foo.com/sa/admin1", "spiffe://foo.com/sa/admin2"] is 
  authorized to access any RPC methods in pkg.service
- Any authenticated user is allowed to access "foo" and "bar" RPC methods if the
  HTTP header includes a name "dev-path" with prefix value "dev/path/".
- Nobody can access "secret" RPC method, not even the admins as deny rules are evaluated 
  first.

```json
{
	"name": "example-policy",
	"allow_rules": [
		{
			"name": "admin-access",
			"source": {
				"principals": [
					"spiffe://foo.com/sa/admin1",
					"spiffe://foo.com/sa/admin2"
				]
			},
			"request": {
				"paths": ["/pkg.service/*"]
			}
		},
		{
			"name": "dev-access",
			"source": {
				"principals": []
			},
			"request": {
				"paths": [
					"/pkg.service/foo",
					"/pkg.service/bar"
				],
				"headers": [
					{
						"key": "dev-path",
						"values": ["/dev/path/*"]
					}
				]
			}
		}
	],
	"deny_rules": [
		{
			"name": "deny-access",
			"request": {
				"paths": [
					"*/secret"
				]
			}
		}
	]
}
```

### API

gRPC will support both static initialization and dynamically reloading the policy
from filesystem. In static initialization, the policy will be provided as a JSON
string. In dynamic file reloading, the application will specify the file path that
contains the authorization policy in JSON format.

Valid user provided SDK authorization policy creates authorization engine(s).
In the case of file watcher, we internally create thread(C-core)/ goroutine(Go)/
scheduled service(Java) which will be used to read the policy file periodically,
and update the authorization engines. During the first file read, if the policy
is invalid or there are I/O errrors, we will return error back to application and
SDK authorization won't be enabled. If the error occurs on a later reload, then
that particular reload will be skipped and error will be logged, and we will
continue to use the latest valid policy to make authorization decisions.

For each incoming RPC request, we will invoke the Engines (Deny engine followed
by Allow engine), to get the authorization decision. We use a C-core filter for
C++, and interceptors for Java and Go.

We recommend users to use a single SDK authorization policy per gRPC server. If
there are multiple policies, then there is a possibility that all the policies
may not be evaluated against. For ex. if we have two policies for two different
services say service A and service B. RPC to service B may get rejected, without
even evaluating against service B policy, because it is evaluated after service A
policy. On getting no match, service A policy could deny by default.

Following code snippets show how to enable authorization in gRPC servers in
different languages.

#### C++

1. C++ authorization provider API

```C++
// Wrapper around C-core grpc_authorization_policy_provider. Internally, it
// handles creating and updating authorization engine objects, using SDK
// authorization policy.
class AuthorizationPolicyProviderInterface {
 public:
  virtual ~AuthorizationPolicyProviderInterface() = default;
  virtual grpc_authorization_policy_provider* c_provider() = 0;
};

// Implementation obtains authorization policy from static string. This provider
// will always return the same authorization engines.
class StaticDataAuthorizationPolicyProvider
      : public AuthorizationPolicyProviderInterface {
 public:
  static std::unique_ptr<StaticDataAuthorizationPolicyProvider>
     Create(const std::string& authz_policy, grpc::Status* status);

  ~StaticDataAuthorizationPolicyProvider override();
 private:
  grpc_authorization_policy_provider* provider_;
};

// Implementation obtains authorization policy by watching for changes in
// filesystem. This provider will return up-to-date authorization engines.
class FileWatcherAuthorizationPolicyProvider final
    : public AuthorizationPolicyProviderInterface {
 public:
  static std::unique_ptr<FileWatcherAuthorizationPolicyProvider>
    Create(const std::string& authz_policy_path,
           unsigned int refresh_interval_sec,
           grpc::Status* status);

  ~FileWatcherAuthorizationPolicyProvider override();
 private:
  grpc_authorization_policy_provider* provider_;
};
```

2. C-core APIs

```C++
/** Channel args for grpc_authorization_policy_provider. If present, enables gRPC
 *  authorization check. */
#define GRPC_ARG_AUTHORIZATION_POLICY_PROVIDER "grpc.authorization_policy_provider"

/**
 * An opaque type that is responsible for providing authorization policies to
 * gRPC.
 */
typedef struct grpc_authorization_policy_provider grpc_authorization_policy_provider;

/**
 * Creates a grpc_authorization_policy_provider using SDK authorization policy
 * from static string.
 * - authz_policy is the input SDK authorization policy.
 * - code is the error status code on failure. On success, it equals
 *   GRPC_STATUS_OK.
 * - error_details contains details about the error if any. If the
 *   initialization is successful, it will be null. Caller must use gpr_free to
 *   destroy this string.
 */
GRPCAPI grpc_authorization_policy_provider* grpc_authorization_policy_provider_static_data_create(
    const char* authz_policy, grpc_status_code* code, const char** error_details);

/**
 * Creates a grpc_authorization_policy_provider using SDK authorization policy
 * from filesystem.
 * - authz_policy_path is the file path of SDK authorization policy.
 * - refresh_interval_sec is the refreshing interval that we will check the
 *   policy file for updates.
 * - code is the error status code on failure. On success, it equals
 *   GRPC_STATUS_OK.
 * - error_details contains details about the error if any. If the
 *   initialization is successful, it will be null. Caller must use gpr_free to
 *   destroy this string.
 */
GRPCAPI grpc_authorization_policy_provider* grpc_authorization_policy_provider_file_watcher_create(
	const char* authz_policy_path, unsigned int refresh_interval_sec, 
	grpc_status_code* code, const char** error_details);

/**
 * Releases grpc_authorization_policy_provider object. The creator of
 * grpc_authorization_policy_provider is responsible for its release.
 */
GRPCAPI void grpc_authorization_policy_provider_release(grpc_authorization_policy_provider* provider);

```

3. Application enables SDK authorization in gRPC Server

- Policy is statically initialized

```C++
grpc::Status status;
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	StaticDataAuthorizationPolicyProvider::Create(authz_policy, &status);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

- Policy is reloaded dynamically from file path

```C++
grpc::Status status;
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	FileWatcherAuthorizationPolicyProvider::Create(
		authz_policy_path, /*refresh_interval_sec=*/3600, &status);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

#### Go

1. Authorization Server Interceptors

```Go
package authz

// StaticInterceptor contains engines used to make authorization decisions.
type StaticInterceptor struct {
  engines ChainEngine
}

// NewStatic returns a new StaticInterceptor from a static authorization policy
// JSON string.
func NewStatic(authzPolicy string) (*StaticInterceptor, error) {
  // ...
}

// UnaryInterceptor intercepts incoming Unary RPC requests.
// Only authorized requests are allowed to pass. Otherwise, an unauthorized
// error is returned to the client.
func (i *StaticInterceptor) UnaryInterceptor(
	ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler)
		(resp interface{}, err error) {
  // ...
}

// StreamInterceptor intercepts incoming Stream RPC requests.
// Only authorized requests are allowed to pass. Otherwise, an unauthorized
// error is returned to the client.
func (i *StaticInterceptor) StreamInterceptor(
	srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler)
		(err error) {
  // ...
}

type FileWatcherInterceptor struct {
  internalInterceptors StaticInterceptors
  policyFile           string
  policyContents       string
  refreshDuration      time.Duration
  cancel               context.CancelFunc
}

// NewFileWatcher returns a new FileWatcherInterceptor from a policy file
// that contains JSON string of authorization policy and a refresh duration to
// specify the amount of time between policy refreshes.
func NewFileWatcher(file string, duration time.Duration) (*FileWatcherInterceptor, error);

// run is a long running goroutine which watches for changes in file path, and
// updates the internalInterceptors upon modification.
func (i *FileWatcherInterceptor) run(ctx context.Context) {
  // ...
}

// updateInternalInterceptors checks if the policy file that is watching has changed,
// and if so, updates the internalInterceptors with the policy. Unlike the
// constructor, if there is an error in reading the file or parsing the policy, the
// previous internalInterceptors will not be replaced.
func (i *FileWatcherInterceptor) updateInternalInterceptors() {
  // ...
}

func (i *FileWatcherInterceptor) UnaryInterceptor(
	ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) 
		(resp interface{}, err error) {
  // ...
}

func (i *FileWatcherInterceptor) StreamInterceptor(
	srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) 
		(err error) {
  // ...
}

// Close cleans up resources allocated by the interceptors.
func (i *FileWatcherInterceptor) Close() {
  i.cancel()
}
```

2. Application enables SDK authorization in gRPC Server

- Policy is statically initialized

```Go
creds := credentials.NewServerTLSFromFile(certFile, keyFile)
i, err := authz.NewStatic(authzPolicy)
serverOpts := []grpc.ServerOption{
  grpc.Creds(creds),
  grpc.ChainUnaryInterceptor(i.UnaryInterceptor),
  grpc.ChainStreamInterceptor(i.StreamInterceptor),
}
s := grpc.NewServer(serverOpts...)
```

- Policy is reloaded dynamically from file path

```Go
creds := credentials.NewServerTLSFromFile(certFile, keyFile)
i, err := authz.NewFileWatcher(authzPolicyFile, 1 * time.Hour)
serverOpts := []grpc.ServerOption{
  grpc.Creds(creds),
  grpc.ChainUnaryInterceptor(i.UnaryInterceptor),
  grpc.ChainStreamInterceptor(i.StreamInterceptor),
}
s := grpc.NewServer(serverOpts...)
// In the end, free up the resources used for policy refresh.
defer i.Close()
```

#### Java

1. Authorization Server Interceptors

```Java
package io.grpc.authz;

// Class of authorization server interceptor for static policy.
public final class AuthorizationServerInterceptor implements ServerInterceptor {
  // Constructor
  private AuthorizationServerInterceptor(String authorizationPolicy) {
    // Creates authorization engines. An IllegalArgumentException will be thrown
    // if the policy file is invalid.
  }

  @Override
  public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
      ServerCall<ReqT, RespT> call, Metadata headers,
      ServerCallHandler<ReqT, RespT> next) {
    // ...
  }

  public static AuthorizationServerInterceptor create(String authorizationPolicy)
      throws IllegalArgumentException {
    return new AuthorizationServerInterceptor(authorizationPolicy);
  }
}

// Class of authorization server interceptor for policy from file with refresh
// capability.
public final class FileAuthorizationServerInterceptor implements ServerInterceptor {
  private volatile AuthorizationServerInterceptor internalAuthzServerInterceptor;
  private final String policyFile;
  private FileTime lastModifiedTime;

  // Constructor
  private FileAuthorizationServerInterceptor(File policyFile) {
    // Read policy from policyFile and create an internalAuthzServerInterceptor. An
    // IOException will be thrown if the policy file cannot be read initially or
    // parsed correctly.
  }

  @Override
  public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
      ServerCall<ReqT, RespT> call, Metadata headers,
      ServerCallHandler<ReqT, RespT> next) {
    return internalAuthzServerInterceptor.interceptCall(call, headers, next);
  }

  // Check if the policy file has been modified, if so, read authorization policy
  // from the policy file and create a new internalAuthzServerInterceptor.
  // Unlike the constructor, IOException here will be caught and logged and the
  // previous internalAuthzServerInterceptor will be continuously used.
  void checkAndReloadPolicy();

  // Closeable for scheduling policy refreshes.
  public Closeable scheduleRefreshes(
      long delay, TimeUnit unit, ScheduledExecutorService executor) {

    final ScheduledFuture<Void> future =
        executor.scheduleWithFixedDelay(delay, unit, new Runnable() {
          @Override public void run() {
            checkAndReloadPolicy();
          }
        });

    return new Closeable() {
      @Override public void close() {
        future.cancel(false);
      }
    };
  }

  public static FileAuthorizationServerInterceptor create(File policyFile)
      throws IOException {
    return new FileAuthorizationServerInterceptor(policyFile);
  }
}
```

2. Application enables SDK authorization in gRPC Server

- Policy is statically initialized

```Java
AuthorizationServerInterceptor authzServerInterceptor;
try {
  authzServerInterceptor = AuthorizationServerInterceptor.create(authzPolicy);
} catch (IllegalArgumentException iae) {
  // ...
}
Server server =
    Grpc.newServerBuilderForPort(port, serverCreds)
        .addService(service)
        .intercept(authzServerInterceptor)
        .build()
        .start();
```

- Policy is reloaded dynamically from file path

```Java
ScheduledExecutorService scheduledExecutor =
    Executors.newSingleThreadScheduledExecutor(
        new ThreadFactoryBuilder()
            .setNameFormat(...)
            .setDaemon(true)
            .build());
FileAuthorizationServerInterceptor authzServerInterceptor;
try {
  authzServerInterceptor = FileAuthorizationServerInterceptor.create(authzPolicyFile);
} catch (IllegalArgumentException iae) {
  // ...
} catch (IOException ioe) {
  // ...
}
Closeable closeable = authzServerInterceptor.scheduleRefreshes(
    1, TimeUnit.HOURS, scheduledExecutor);
Server server =
    Grpc.newServerBuilderForPort(port, serverCreds)
        .addService(service)
        .intercept(authzServerInterceptor)
        .build()
        .start();
// In the end, free up the resources used for policy refresh.
closeable.close();
```

## Rationale

gRPC Authorization internally implements RBAC Engine(s) based on Envoy [RBAC policy](https://github.com/envoyproxy/envoy/blob/main/api/envoy/config/rbac/v3/rbac.proto).
We decided to create a new policy language "SDK authorization policy" instead of 
consuming Envoy RBAC directly due to following reasons:
- Envoy RBAC is a complex language, and we preferred using a simple human readable 
  policy language
- With our own language, we can provide a stable API, even when Envoy undergoes 
  versioning updates.

Note that SDK authorization policy is a subset of Envoy RBAC, and it does not
support all the fields that are present in Envoy RBAC. 

RBAC policy provides service-level and method-level access control for a service.
Engines process incoming RPC request attributes against policy configs and make a
decision on whether to allow or deny the request. The decision depends on the type
of policy (if the policy is an allowlist or denylist) and whether a matching policy
was found.

This engine implementation is shared by SDK-based authorization and xDS-based
authorization. In gRPC Proxyless Service Mesh (or xDS authorization), Istio or
Traffic Director control plane sends Envoy RBAC policy configs to xDS enabled gRPC
servers for authorization. ([A41](https://github.com/grpc/proposal/pull/237): xDS RBAC Support)

Overall SDK authorization flow is as follows. User supplies gRPC SDK authorization
policy to provider/interceptor. The provider then forwards the JSON policy to Policy
translator. The translator converts JSON policy to Envoy RBAC protos (Allow and/or
Deny policy). Translator errors out on I/O errors or if the policy does not
represent JSON schema it currently supports. Ultimately the generated RBAC policies
are used to create Envoy RBAC authorization engine(s). Then, for each incoming RPC
request, we will invoke the Engines to get the authorization decision.

SDK authorization can be enabled in xDS enabled servers, which means both
authorization paths (SDK and xDS authorization) can co-exist. For a request to
be allowed, both paths must allow the request.

As mentioned previously, authorization APIs take policy in JSON format, instead of
protobuf. This is done to avoid the dependency on protobuf. We have users in OSS
that use gRPC without protobuf. Another reason is to have a consistent API across
languages.

## Implementation

The implementation order will be C++, Go, Java and then wrapped languages. The 
implementation will be done by ashithasantosh.
