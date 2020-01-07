gRPC-Web in-process proxy and client for grpc-dotnet
----
* Author(s): James Newton-King
* Approver: Jan Tattermusch & Wenbo Zhu
* Status: Approved
* Implemented in: https://github.com/grpc/grpc-dotnet/pull/695
* Last updated: 2022-01-4
* Discussion at: TBD

## Abstract

gRPC-Web for grpc-dotnet adds:

* An in-process proxy for gRPC hosted on ASP.NET Core, allowing the server to support gRPC-Web calls.
* A gRPC-Web outgoing proxy for the .NET gRPC client, allowing the client to make gRPC-Web calls.

## Background

gRPC requires HTTP/2 features that are not available in browser HTTP clients. [gRPC-Web](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md) modifies the gRPC specification to allow gRPC to be callable from the browser.

Traditionally gRPC-Web support is added to a service via a proxy. Browsers call the proxy and the proxy translates the gRPC-Web call to gRPC HTTP/2 and forwards it on to the service implementation. This works, but there is effort, complexity and performance costs associated with adding a proxy to a solution. An in-process proxy allows for gRPC-Web to be supported by adding additional packages and startup configuration to an existing app. Being able to support gRPC-Web with an in-process proxy removes many of the costs of the traditional gRPC-Web proxy.

Calling gRPC-Web is typically done using the JavaScript gRPC-Web library. This works for JavaScript solutions, but the modern .NET ecosystem allows for .NET applications to run in the browser using Web Assembly. A gRPC-Web .NET client is needed for this scenario.

The grpc-dotnet gRPC server is built on ASP.NET Core and hosted on Kestrel, a full-featured, multi-purpose HTTP/1.1 and HTTP/2 server. ASP.NET Core has a [request pipeline that an application can plug middleware into](https://docs.microsoft.com/aspnet/core/fundamentals/middleware) to process the request and response. ASP.NET Core also supports configuring CORS for an application.

The grpc-dotnet gRPC client is built on HttpClient, a full-featured, multi-purpose HTTP/1.1 and HTTP/2 client. HttpClient has an [outgoing request pipe that an application can plug handlers into](https://docs.microsoft.com/dotnet/api/system.net.http.httpmessagehandler).

## Proposal

* Add `Grpc.AspNetCore.Web`, a new package that acts as a in-process proxy for gRPC-Web. This package provides middleware via `UseGrpcWeb()` that detects gRPC-Web requests by their content-type, and translates them to gRPC HTTP/2. Request headers and the request message(s) are translated inbound, and response headers/footers and response message(s) are translated back to gRPC-Web outbound. Base64 content is encoded and decoded as content is read. There is configuration to control which services support gRPC-Web. A developer can choose to enable gRPC-Web for all services in `AddGrpcWeb`, or opt-in and opt-out on individual services.

*Startup.cs* configuration file:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddGrpc();

    // Enable gRPC-Web for all services
    services.AddGrpcWeb(o => o.GrpcWebEnabled = true);
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();

    // Add gRPC-Web middleware
    app.UseGrpcWeb();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapGrpcService<GreeterService>();
    });
}
```

* Add `Grpc.Net.Client.Web`, a new package that acts as a gRPC-Web outgoing proxy for the .NET gRPC client. This package provides a `DelegatingHandler` that translates outgoing gRPC HTTP/2 requests to gRPC-Web. Base64 content is encoded and decoded as content is read. When a gRPC channel is created the developer can choose to configure it to use a HttpClient that has the `GrpcWebHandler`. When the handler is created the developer chooses whether to use `application/grpc-web` or `application/grpc-web-text`.

```csharp
// Create delegating handler and configure to use application/grpc-web-text
var grpcWebHandler = new GrpcWebHandler(GrpcWebMode.GrpcWebText, new HttpClientHandler());

var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
    {
        HttpClient = new HttpClient(grpcWebHandler)
    });
var client = new Greeter.GreeterClient(channel);
```

### Interoperability

It is critical that any new gRPC-Web client and server implementations are compatible with existing gRPC-Web implementations.

For interop testing, the [gRPC interop test suite](https://github.com/grpc/grpc/blob/master/doc/interop-test-descriptions.md) is executed using grpc-web. The existing grpc-dotnet interop test server and test client have been updated with configuration options to use grpc-web.

Some tests in the test suite can't be run using gRPC-Web (i.e. some auth tests, client and bidirectional streaming tests) and are excluded.

The tests used for gRPC-Web interop testing are:

* empty_unary
* large_unary
* server_streaming (grpc-web-text only)
* custom_metadata
* special_status_message
* unimplemented_service
* unimplemented_method
* client_compressed_unary
* server_compressed_unary
* server_compressed_streaming (grpc-web-text only)
* status_code_and_message

## Release

A stable implementation was released with version 2.29.0 in May 2020.

## Implementation
All the relevant details are described in the proposal section above.
