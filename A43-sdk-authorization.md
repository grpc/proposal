A43: SDK authorization

* Author(s): [Ashitha Santhosh](https://github.com/ashithasantosh)
* Approver: 
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-06-17
* Discussion at: (filled after thread exists)

## Abstract

gRPC is currently implementing an open-source standard authorization solution
based on Envoy RBAC. This proposal discusses how this framework would be made 
available to non-xDS OSS gRPC users to perform per-RPC authorization checks.

## Background

Authentication verifies the authority of the request. Authorization ensures the 
authenticated authority has access to the requested resource.

gRPC offers built-in authentication mechanisms like TLS, ALTS and even allow
users to plug in their own authentication systems. However, gRPC doesn't have any
standard authorization solution available. If a user wanted to perform per RPC
authorization check, they have to implement their own solution in the application.
With standard gRPC authorization solution (under implementation), users can simply
provide a policy configuration for their service, then the server verifies
whether the client is authorized to access the requested service, and for which
RPC methods. This differs from client-side authorization (also known as hostname
verification) where client verifies server's identity.

We have two different use-cases for authorization-
1. gRPC Proxyless Service Mesh (or xDS authz) - Here Istio or Traffic Director
   control plane sends Envoy RBAC policy configs to xDS enabled gRPC servers for
   authorization
2. SDK authorization (or non-xDS authz) - Authorization is made available to
   non-xDS OSS gRPC users like network switches.

This document discusses details of SDK authorization use-case. xDS authorization
will be convered in a different proposal.

## Related Proposals:

* [A41](https://github.com/grpc/proposal/pull/237): xds RBAC Support

## Proposal

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
			    "in the form of \"/package.service/method\". The paths are ORed together, i.e., it"
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
					  "headers are *not* supported: \"hop-by-hop\" headers (e.g.,"
					  "those listed in \"Connection\" header), the \"Host\" header,"
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
						"- Prefix match: \"abc*\" will match on value \"abc\" and \"abcd\"."
						"- Suffix match: \"*abc\" will match on value \"abc\" and \"xabc\"."
						"- Presence match: \"*\" will match when the value is not empty.",
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

#### Details

gRPC SDK authorization policy has a list of allow rules and a list of deny rules. 
The following sequence is followed to make an authorization decision -
1. Check for a match in deny rules, if a match is found, the request will be denied. 
   If no match is found in deny rules, or there are no deny rules execute next step.
2. Check for a match in allow rules, if a match is found, the request will be allowed. 
   Note that allow rules is a required field here, if not present, policy is invalid.
3. If no match is found in allow rules, we deny the request.

Each rule has the following semantics -
1. Each rule has a name, a source and a request to match against. For a rule to match,
   both source and request must match.
   - If both source and request are empty, and the policy is an allow list, we will
     accept all RPC requests. On the other hand, if the policy were a denylist, we
     would deny all RPC requests.
   - If only source is empty, we evaluate against the request fields and apply that to
     any user (See example below). Similarly, if only request is empty, we evaluate
     against the source fields and apply that to any action.
   - This also applies to sub fields, like if the principals in source is empty, the
     behavior would be similar to an empty source.
2. Each source could contain a list of principals. The principals are ORed together,
   i.e. it matches if one of them matches.
   Sequence of steps to evaluate each principal from config -
   1. If TLS is not used, matching fails.
   2. A match is found if principal matches any one of URI SANs or DNS SANs or Subject
      field from x509 client certificate.
   3. We also get a match if there is no client certificate, and principal is an empty
      string.
3. Each request could contain a list of URL paths (i.e. fully qualified RPC methods)
   and list of http headers to match. Refer JSON schema above to understand matching
   semantics.

#### Example

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

### API

gRPC will support both static initialization and dynamically reloading the policy
from filesystem. In static initialization, the policy will be provided as a JSON
string. In dynamic file reloading, the application will specify the file path that
contains the authorization policy in JSON format.

We recommend the users to use a single SDK authorization policy per gRPC server.
If there are multiple policies, then there is a possibility that all the policies
may not be evaluated against. For ex. if we have two policies for two different
services say service A and service B. RPC to service B may get rejected, without
even evaluating against service B policy, because it is evaluated after service A
policy. On getting no match, service A policy could deny by default.

Following code snippets show how to enable authorization in gRPC servers in 
different languages.

#### C++

1. Static Initialization

```C++
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	StaticDataAuthorizationPolicyProvider::Create(authz_policy);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

2. Dynamic file reloading

```C++
std::shared_ptr<AuthorizationPolicyProviderInterface> provider = 
	FileWatcherAuthorizationPolicyProvider::Create(authz_policy_path, /*refresh_interval_sec=*/3600);
ServerBuilder builder;
builder.SetAuthorizationPolicyProvider(provider);
std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
```

#### Go

1. Static initialization

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

2. Dynamic File Reloading

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

1. Static initialization

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
        .intercept(authzServerInterceptor);
        .build()
        .start();
```

2. Dynamic file reloading

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

### High level Implementation

gRPC Authorization internally implements RBAC Engines based on Envoy RBAC policies.
RBAC policy provides service-level and method-level access control for a service.
Engines process incoming RPC request attributes against policy configs and make a
decision on whether to allow or deny the request. The decision depends on the type
of policy (if the policy is an allowlist or denylist) and whether a matching policy
was found. Engine implementation is shared by xDS and SDK authorization.

[RBAC filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/http/rbac/v3/rbac.proto

In SDK authorization, user supplies gRPC SDK authorization policy to policy provider
interface. In the case of file watcher, the provider is responsible for initializing
the thread(C++)/ goroutine(Go)/ scheduled service(Java) which will be used to read
the policy file periodically. The provider then forwards the JSON policy to Policy
translator. The translator converts JSON policy to Envoy RBAC protos (Allow and/or
Deny policy). Note that SDK authorization policy is a subset of Envoy RBAC, and it
does not support all the fields that are present in Envoy RBAC. Ultimately the
generated RBAC policies are used to create Envoy RBAC engine(s).

For each incoming RPC request, we will invoke the Evaluate functionality in Engines
(Deny engine followed by Allow engine), to get the authorization decision. We use a
C-core filter for C++, and interceptors for Java and Go.

## Rationale

We decided to create a new policy language "SDK authorization policy" instead of 
consuming Envoy RBAC directly due to following reasons
- Envoy RBAC is a complex language, and we preferred using a simple human readable 
  policy language
- With our own language, we can provide a stable API, even when Envoy undergoes 
  versioning updates.

Authorization APIs take policy in JSON format, instead of protobuf. gRPC wants to
avoid the dependency on protobuf when possible. We have users in OSS that use gRPC
without protobuf. Another reason is to have a consistent API across languages.

## Implementation

The implementation order will be C++, Go, Java and then wrapped languages. The 
implementation will be done by ashithasantosh.
