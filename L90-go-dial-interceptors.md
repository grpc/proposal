L90: Support global dial interceptors in Golang
----
* Author(s): Anirudh Ramachandra
* Approver: dfawley
* Status: Draft
* Implemented in: Go
* Last updated: 2021-11-16
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Support dial interceptors in grpc-go which allow users to apply business logic globally to all clients including those created in the gRPC package.

## Background

grpc.Dial is usually the entry point for all client operations. It is possible to customize the behavior of grpc.Dial using dial options including interceptors. When dial options are not good enough, teams are free to create a wrapper Dial function and use it as the entry for their clients. Popular use cases include adding business logic to track statistics, making tracing easy and applying custom socket options. The wrapper Dial usually provides hooks to apply the necessary business logic before finally calling grpc.Dial.

An issue arises when the gRPC package itself is responsible for creating clients. For example in the xDS layer, a gRPC client connection is established with the xDS server. As the gRPC package itself uses grpc.Dial and cannot use the wrapper Dial it is not possible to apply the business logic.

This RFC proposes a new global dial interceptor in Golang to provide a cleaner way to add these hooks to all clients including those created within the gRPC library.

### Related Proposals:
N/A

## Proposal

gRPC will support setting a new global Dial interceptor which if set will be called as part of every grpc.Dial/grpc.DialContext. The application can set the dial interceptor as part of its initialization by calling "SetDialInterceptor"

```
type dialerCallback func(ctx context.Context, target string, opts ...DialOption) (*ClientConn, error)

type DialInterceptor func(ctx context.Context, target string, defaultDial dialerCallback, opts ...DialOption) (*ClientConn, error)

var dialInterceptor DialInterceptor

func SetDialInterceptor(interceptor DialInterceptor) {
  dialInterceptor = interceptor
}
```

Internally, the gRPC DialContext will be modified to call the dial interceptor if it's set. If the dial interceptor is not set, the current behavior is maintained. Global Dial interceptors will also receive the functor to grpc.dialContext which can be called after performing the required application logic.

```
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
  if dialInterceptor == nil {
    return dialContext(ctx, target, opts...)
  }
  return dialInterceptor(ctx, target, dialContext, opts...)
}

func dialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
  // Same logic as that existed before in DialContext.
}
```

## Rationale

Supporting a dial interceptor allows users of gRPC go to inject their business logic while also handling clients that are implicitly created in the gRPC package. Currently, this is only an issue for gRPC clients created within the gRPC package i.e. xDS but longer term this will be a major pain point for users who have custom wrappers around grpc.Dial.

The gRPC dial interceptor allows users to both inject custom logic as well as gRPC client specific dial options.

An alternative approach to solve this problem is to expose a way of injecting Global Dial Options in clientconn.go.

### Global Dial Options

Similar to the dial interceptor, the global dial options can be set by the application as part of its initialization.
```
// clientconn.go

var globalDialOptions grpc.DialOption

// AddGlobalDialOptions adds options that are applied globally to all
// Dial calls.
func AddGlobalDialOptions(options ...DialOption) {
   globalDialOptions = append(globalDialOptions, options...)
}
```

grpc.DialContext can then be augmented so that the global options are considered for each client that is created. This can be done by a simple prepend operation with DialContext opts params.
```
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
  // Apply globalDialOptions to Dial calls.
  opts = append(globalDialOptions, opts...)
  ...
}
```

#### Pros

* This approach is simpler as it restricts the possible configurations that can be configured as part of the application

#### Cons

* Does not allow any manipulation of the target or the context.
* Manipulating the dial options is not enough to apply most application logic.


While this use case might be able to handle *most* use cases where custom logic can be injected using interceptors, it is restrictive compared to the dial interceptor. The actual work needed for either of these options is similar, so it makes sense to converge on the Dial Interceptor.

## Implementation

* Support a new gRPC Dial interceptor which is similar to gRPC.DialContext, with gRPC.DialContext as a special parameter for callback.
* Support a new function in clientconn.go to set this global interceptor.
* Any teams that have custom wrappers that finally call grpc.Dial, can then shift their business logic to the interceptor.