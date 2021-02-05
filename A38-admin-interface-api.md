Admin Interface API
----
* Author(s): lidiz
* Approver: markdroth
* Status: In-Review
* Implemented in: C++, Golang, Java
* Last updated: 2021-02-05
* Discussion at: https://groups.google.com/g/grpc-io/c/0B2f9ha5HoM

## Abstract

This proposal describes a convenience API in each gRPC language to improve the usability of creating a gRPC server with admin services to expose states in the gRPC library.


## Background

### Services Exposing gRPC Internal States

gRPC provides a set of official gRPC services that serve gRPC internal states. This proposal will focus on the ones that expose the state of the entire gRPC library, which is agnostic to which gRPC server they were bound to:

* Channelz: the channel tracing library that collects stats about a channel’s traffic, underlying sockets, and credentials ([gRFC A14](https://github.com/grpc/proposal/blob/master/A14-channelz.md));
* [Client Status Discovery Service](https://github.com/envoyproxy/envoy/blob/main/api/envoy/service/status/v3/csds.proto): the xDS config dump library that dumps the accepted xDS configuration with config status;


## Proposal

The admin interface API is a set of API that registers available admin services to a given gRPC server. There will be more gRPC services boosting debuggability of gRPC down the road. With this API, applications will automatically get the update as they pulling in the latest gRPC library.

There are several different options for the convenience API design. Each option is a tradeoff between simplicity to use and flexibility. The simplicity to use is defined by lines of code to create a serving gRPC server with admin services, and the additional cognitive cost to learn the new API or manage dependencies. The flexibility is defined by how can users provide additional server creation options. To date, users can specify one or more **listening addresses, credentials, TCP/HTTP config, codec, interceptors and service implementations** during the server creation process. Not to mention the potential  impact on the threading model during the start of the server. If we want to bundle as many options as possible into one line of code, we will inevitably sacrifice some flexibility. For other alternative designs, see section below.

This proposal recommend the admin interface API design to be **a function that combines admin services from different packages**.

Here are the usage snippets of admin interface API in each language:


### Java

#### Create An Admin Server With Existing gRPC Java API:

```java
server = ServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .addService(new ChannelzService())
        .addService(new CsdsService())
        .build()
        .start();
server.awaitTermination();
```


#### Create An Admin Server With gRPC Java Admin Interface API:

```java
server = ServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .addServices(AdminInterface.getStandardServices())
        .build()
        .start();
server.awaitTermination();
```


### Golang

#### Create An Admin Server With Existing gRPC Golang API:

```golang
lis, err := net.Listen("tcp", ":50051")
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
defer lis.Close()
grpcServer := grpc.NewServer()
channelzService.RegisterChannelzServiceToServer(grpcServer)
csdsService.RegisterCsdsServiceToServer(grpcServer)
if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
}
```


#### Create An Admin Server With gRPC Golang Admin Interface API:

```golang
lis, err := net.Listen("tcp", ":50051")
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
defer lis.Close()
grpcServer := grpc.NewServer(...opts)
adminServices.RegisterAdminServicesToServer(grpcServer)
if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
}

```


### C++

#### Create An Admin Server With Existing gRPC C++ API:

```cpp
grpc::ServerBuilder builder;
builder.RegisterService(grpc::ChannelzService());
builder.RegisterService(grpc::CsdsService());
builder.AddListeningPort(":50051", grpc::ServerCredentials(...));
std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
```


#### Create An Admin Server With gRPC C++ Admin Interface API:

The admin services will use the gRPC C++ Sync API to offload the responsibility of managing polling and thread from users. In future, they might migrate to the gRPC C++ Callback API.

```cpp
grpc::ServerBuilder builder;
grpc::AddAdminServices(&builder);
builder.AddListeningPort(":50051", grpc::ServerCredentials(...));
std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
```

### Dependency-based Admin Services List

In addition, by providing new APIs, we abstract away the individual services with the concept admin services. We can also achieve **dynamic admin services selection**, for example, we **only add CSDS service if the xDS package is pulled in as a dependency of the application**. In this way, the application can avoid adding unnecessary dependencies.


## Rationale

### Start Admin Server without Code Change

In other softwares, like Envoy, it’s common to see a pattern that specifying an environment variable will instruct the application to open a debugging/administration site. Or, for a more meticulous library, users can specify the admin configuration in their config files. However, those solutions don't apply to gRPC:

1. The default unsecured pattern has caused endpoint security concern in Envoy (see [envoy#2763])(https://github.com/envoyproxy/envoy/issues/2763));
2. gRPC as a library should not open additional listening port without explicitly application code.


### Alternative Admin Interface API Designs

Here are the usage snippets for other API Designs in Java:

#### A method of the gRPC server class that adds admin services
```java
server = ServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .addAdminService()
        .build()
        .start();
server.awaitTermination();
```

The drawback for this option is dependency management. To date, the gRPC official admin services Channelz/Reflection/Health are distributed as separated package to keep the main package focus as a network library. If we apply this option, then we will void the separate effort.


#### A function that creates the gRPC server object bound with admin services;
```java
server = AdminServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .build()
        .start();
server.awaitTermination();
```

For this option, the `AdminServerBuilder` allows some flexibility, but it conflicts with `XdsServerBuilder` if users want to enable xDS, and it would be repetitive if we offers `AdminXdsServerBuilder` on the side.


#### A function that creates and starts the gRPC server with admin services.
```java
server = admin.StartAdminServer(50051, certChainFile, privateKeyFile)
server.awaitTermination();
```

For this option, advanced users might want to play with the admin service like adding interceptors or change credential types, but even tiny amount of change means fallback to use existing API.


## Future Work

### Console-to-Console Credentials

To date, if we want to securely guard an endpoint, the recommended way is using a TLS cert or via xDS’s mTLS (if rolled out). With proper setup, they work fine for long-lasting debugging ports. However, both of them are overly complicated for ad-hoc debugging, and may unfortunately nudge people to use insecure connections. From users’ perspective, they just want to inspect an gRPC application in one console with the tool in another console.

The problem has been solved by many other softwares, like Grafana uses username and password, iPython notebook prints a generated auth token to the console. The admin server serves sensitive traffic statistics that should be protected. With easy-to-use credentials, we are promoting protected connections, and it will prevent gRPC from following the Envoy’s admin interface design.


## Implementation

TBD
