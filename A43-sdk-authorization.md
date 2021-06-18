A43: SDK authorization

Author(s): [Ashitha Santhosh](https://github.com/ashithasantosh)
Approver: 
Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
Implemented in: <language, ...>
Last updated: 2021-06-17
Discussion at: (filled after thread exists)

##Abstract

gRPC is currently implementing an open-source standard authorization solution
based on Envoy RBAC. This proposal discusses how this framework would be made 
available to OSS gRPC users to perform per-RPC authorization checks.

##Background

gRPC authorization support which is currently under implementation, internally 
implements RBAC Engines based on Envoy RBAC policies. RBAC policy provides 
service-level and method-level access control for a service. Engines process 
incoming RPC request attributes against policy configs and make a decision on 
whether to allow or deny the request. The decision depends on the type of policy 
(if the policy is an allowlist or denylist) and whether a matching policy was found.

[RBAC filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/http/rbac/v3/rbac.proto

The RBAC engine implementation is used by xDS API (i.e. Proxyless Service Mesh, 
when Istio or Traffic Director sends Envoy RBAC policy to xDS enabled gRPC servers 
for authorization) and SDK API (which makes authorization available to OSS gRPC users). 
This document shares details on the latter.

##Related Proposals:

* [A41: xds RBAC Support][A41]

##Proposal

### User facing Authorization Policy

gRPC SDK authorization policy is the user facing policy language that allows
service owners and/or security admins to enable per-RPC authorization checks.

SDK authorization policy guarantees stability and backward compatibility. So if
a user writes a config, it will always be supported by gRPC unless the policy
undergoes major versioning change like adding required fields. The policy is 
extendable, we may add new fields in the future as new requirements are introduced, 
we won’t be removing fields or updating semantics of existing fields.

Following is the JSON schema of SDK Authorization Policy Version 1.0

```json
{
	"title": "AuthorizationPolicy",
	"definitions": {
		"rule": {
			"description": "Specification of rules. An empty rule is always matched (i.e., both source and request are empty)",
			"type" : "object",
			"properties": {
				"name": {
					"description": "The name of an authorization rule. It is mainly for monitoring and error message generation.",
					"type": "string"
				},
				"source": {
					"description": "Specifies attributes of a peer. Fields in the source are ANDed together, once we support" 
					  "multiple fields in the future. If not set, no checks will be performed against the source.",
					"type": "object",
					"properties": {
						"principals": {
							"description": "A list of peer identities to match for authorization. The principals are one"
							  "of, i.e., it matches if one of the principals matches."
							  "The field supports Exact, Prefix, Suffix, and Presence matches."
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
					"description": "Specifies attributes of a request. Fields in the request are ANDed together. If not set, no"
					  "checks will be performed against the request.",
					"type": "object",
					"properties": {
						"paths": {
							"description": "A list of paths to match for authorization. This is the fully qualified name"
							  "in the form of \"/package.service/method\". The paths are""ORed together, i.e., it"
							  "matches if one of the paths matches."
							  "This field supports Exact, Prefix, Suffix, and Presence matches."
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
							"description": "A list of HTTP header key/value pairs to match against, for potentially" 
							  "advanced use cases. The headers are ANDed together, i.e., it matches only if *all*"
							  "the headers match.",
							"type": "array",
							"items": {
								"type": "object",
								"properties": {
									"key": {
										"description": "The name of the HTTP header to match. The following "
										  "headersare *not* supported: \"hop-by-hop\" headers (e.g.,"
										  "those listed in\"Connection\" header), the \"Host\" header,"
										  "HTTP/2 pseudo headers (\":\"-prefixed) and headers prefixed"
										  "with \"grpc-\".",
										"type": "string"
									},
									"values": {
										"description": "A list of header values to match. The header values"
										  "are ORed together, i.e., it matches if one of the values matches."
										  "Multi-valued headers are considered a single value with commas"
										  "added between values."
										  "This field supports Exact, Prefix, Suffix, and Presence match."
											"- Exact match: \"abc\" will match on value \"abc\"."
											"- Prefix match: \"abc*\" will match on value \"abc\" and"
											   "\"abcd\"."
											"- Suffix match: \"*abc\" will match on value \"abc\" and"
											   "\"xabc\"."
											"- Presence match: \"*\" will match when the value is not"
											   "empty.",
										"type": "array",
										"items": {
											"type": "string"
										}
									}
								},
								"required": ["key", "values"],
							}
						}
					}
				}
			},
			"required": ["name"]
		}
	},
	"description": "AuthorizationPolicy defines which principals are permitted to access which resource. Resources are RPC methods"
	  "scoped by services.",
	"type": "object",
	"properties": {
		"name": {
			"description": "The name of an authorization policy. It is mainly for monitoring and error message generation.",
			"type": "string",
		},
		"deny_rules": {
			"description": "List of deny rules to match. If a request matches any of the deny rules, then it will be denied. If none of"
			  "the deny rules matches or there are no deny rules, the allow rules will be evaluated.",
			"type": "array",
			"items": {
				"$ref": "#/definitions/rule",
			},
		},
		"allow_rules": {
			"description": "List of allow rules to match. The allow rules will only be evaluated after the deny rules. If a request"
			  "matches any of the allow rules, then it will allowed. If none of the allow rules matches, it will be denied.",
			"type": "array",
			"items": {
				"$ref": "#/definitions/rule",
			},
		}
	},
	"required": ["name", "allow_rules"]
}
```

gRPC SDK authorization policy has a list of allow rules and a list of deny rules. 
The following sequence is followed to make an authorization decision -
1. Check for a match in deny rules, if a match is found, the request will be denied. 
   If no match is found in deny rules, or there are no deny rules execute next step.
2. Check for a match in allow rules, if a match is found, the request will be allowed. 
   Note that allow rules is a required field here, if not present, policy is invalid.
3. If no match is found in allow rules, we deny the request.

In the following policy example
- Peer identity from ["spiffe://foo.com/sa/admin1", "spiffe://foo.com/sa/admin2"] is 
  authorized to access any RPC methods in pkg.service
- Any authenticated user is allowed to access "foo" and "bar" RPC methods.
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
				"principals": [
					"*"
				]
			},
			"request": {
				"paths": [
					"/pkg.service/foo",
					"/pkg.service/bar"
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

As mentioned previously, internally gRPC Authorization engine consumes Envoy RBAC 
proto. The user provided JSON SDK authorization policy will be translated to Envoy 
RBAC proto and provided to Engines for evaluation. Note that SDK authorization policy 
is a subset of Envoy RBAC, and it does not support all the fields that are present in 
Envoy RBAC.

### API

gRPC will support both static initialization and dynamically reloading the policy
from filesystem. In static initialization, the policy will be provided as a JSON
string. In dynamic file reloading, the application will specify the file path that
contains the authorization policy in JSON format.

Following code snippets show how to enable authorization in gRPC servers in 
different languages.

#### C++

Static Initialization

```C++
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	StaticDataAuthorizationPolicyProvider::Create(authz_policy);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

Dynamic file reloading

```C++
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	FileWatcherAuthorizationPolicyProvider::Create(authz_policy_path, /*refresh_interval_sec=*/3600);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

#### Go

Static initialization

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

Dynamic File Reloading

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

Static initialization

```Java
AuthorizationServerInterceptor authzServerInterceptor;
try {
  authzServerInterceptor = AuthorizationServerInterceptor.create(authzPolicy);
} catch (IllegalArgumentException iae) {
  // ...
}
Server server =
    ServerBuilder.forPort(port)
        .addService(service)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .intercept(authzServerInterceptor);
        .build()
        .start();
```

Dynamic file reloading

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
    ServerBuilder.forPort(port)
        .addService(service)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .intercept(authzServerInterceptor)
        .build()
        .start();
// In the end, free up the resources used for policy refresh.
closeable.close();
```

##Rationale

We decided to create a new policy language "SDK authorization policy" instead of 
consuming Envoy RBAC directly due to following reasons
- Envoy RBAC is a complex language, and we preferred using a simple human readable 
  policy language
- With our own language, we can provide a stable API, even when Envoy undergoes 
  versioning updates.

##Implementation

The implementation order will be C++, Go, Java and then wrapped languages. The 
implementation will be done by ashithasantosh.
