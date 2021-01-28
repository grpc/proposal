Admin Interface API
----
* Author(s): lidiz
* Approver: markroth
* Status: Draft
* Implemented in: C++, Golang, Java
* Last updated: 2021-01-27
* Discussion at: TBD

## Abstract

This proposal describes a convenience API in each gRPC language to improve the usability of creating a gRPC server with admin services.


## Background

gRPC Admin Interface is part of the Proxyless Service Mesh Observability project which includes new gRPC admin services and a CLI tool (yet to release). The admin services are responsible for exposing states of the gRPC library via gRPC protocol, and the CLI tool will help users to better interpret the fetched data from the gRPC application. Currently, the existing admin server should at least expose two admin services:

* Channelz: the channel tracing library that collects stats about a channel’s traffic, underlying sockets, and credentials ([proto definition](https://github.com/grpc/grpc/blob/master/src/proto/grpc/channelz/channelz.proto));
* [Client Status Discovery Service](https://github.com/envoyproxy/envoy/blob/main/api/envoy/service/status/v3/csds.proto): the xDS config dump library that dumps the accepted xDS configuration with config status;

However, the number of admin services might increase in the future. Solutions with no code changes, like env, or config files, are not desired (see rationale section below). On the other hand, as a library, gRPC would like to promote the usage of admin services. So, there is a need for an ideally one LOC to create and start an admin server.

In addition, by providing new APIs, we abstract away the individual services with the concept admin services. We can also achieve **dynamic admin services selection**, for example, we **only add CSDS service if the xDS package is pulled in**. In this way, we won’t force users to take unnecessary dependencies.


### Types of Services

To help communicate the scenarios, we can define all services into several types, from the view of administrative functions:

* **Primary service**: the user-provided services that handle the application logic;
* **Binary-global service**: the services that expose the state of the entire gRPC library, and they serve the same information regardless of which server they have been registered (e.g., Channelz, CSDS);
* **Per-server service**: the services that distribute the state of other registered services on the same server (e.g., Health, Reflection).

The binary-global services contain sensitive information about traffic, configuration, even security, hence we hope users can set up the services securely in our recommended usage or the upcoming new API.

The recommended way of securing the admin service port is binding it to either `localhost` or using a `UDS` port.


## Proposal

There are several different options for the convenience API design to help create a more usable admin interface API. Each option is a tradeoff between simplicity to use and flexibility. The simplicity to use is defined by lines of code to create a serving gRPC server with admin services, and the additional cognitive cost to learn the new API or manage dependencies.

The flexibility is defined by how can users provide additional server creation options. To date, users can specify one or more **listening addresses, credentials, TCP/HTTP config, codec, interceptors and service implementations** during the server creation process. Not to mention the potential  impact on the threading model during the start of the server. If we want to bundle as many options as possible into one line of code, we will inevitably sacrifice some flexibility.

Options to create an admin server are ordered from the most flexible to the least:

1. The control group: using existing API to create the server;
2. A function that combines admin services from different packages; **<-RECOMMENDED BY THIS PROPOSAL**
3. A method of the gRPC server class that adds admin services;
4. A function that creates the gRPC server object bound with admin services;
5. A function that creates and starts the gRPC server with admin services.

This proposal will recommend option 2 as the admin interface API (for other alternatives, see rationale section below). The usage snippet of admin interface API in each language:


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
builder.RegisterService(grpc::experimental::CsdsService());
builder.AddListeningPort(":50051", grpc::ServerCredentials(...));
std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
```


#### Create An Admin Server With gRPC C++ Admin Interface API:

The admin services will use the gRPC C++ Sync API to offload the responsibility of managing polling and thread from users. In future, they might migrate to the gRPC C++ Callback API.

```cpp
grpc::ServerBuilder builder;
grpc::experimental::AddAdminServices(&builder);
builder.AddListeningPort(":50051", grpc::ServerCredentials(...));
std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
```


## Rationale

### Start Admin Server without Code Change

In other softwares, like Envoy, it’s common to see a pattern that specifying an environment variable will instruct the application to open a debugging/administration site. Or, for a more meticulous library, users can specify the admin configuration in their config files. However, those solutions don't apply to gRPC:

1. The default unsecured pattern has caused endpoint security concern in Envoy (see [envoy#2763])(https://github.com/envoyproxy/envoy/issues/2763));
2. gRPC as a library should not open additional listening port without explicitly application code.


### Alternative Admin Interface API Designs

Here are the usage snippets for other API options for Java:

Option 3: Server class method
```java
server = ServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .addAdminService()
        .build()
        .start();
server.awaitTermination();
```

Option 4: Create admin server
```java
server = AdminServerBuilder.forPort(50051)
        .useTransportSecurity(certChainFile, privateKeyFile)
        .build()
        .start();
server.awaitTermination();
```

Option 5: Create and start serving
```java
server = admin.StartAdminServer(50051, certChainFile, privateKeyFile)
server.awaitTermination();
```

For option 4/5, they well achieved the ultimate goal of reducing the line-of-code changes, but they also terrible at satisfying any need for extension. For option 5, advanced users might want to play with the admin service like adding interceptors or change credential types, but even tiny amount of change means fallback to option 1 (existing API). For option 4, the `AdminServerBuilder` allows some flexibility, but it conflicts with `XdsServerBuilder` if users want to enable xDS, and it would be repetitive if we offers `AdminXdsServerBuilder` on the side.

The drawback for option 3 is dependency management. To date, the gRPC official admin services Channelz/Reflection/Health are distributed as separated package to keep the main package focus as a network library. If we apply option 3, then we will void the separate effort.


## Future Work

### Console-to-Console Credentials

To date, if we want to securely guard an endpoint, the recommended way is using a TLS cert or via xDS’s mTLS (if rolled out). With proper setup, they work fine for long-lasting debugging ports. However, both of them are overly complicated for ad-hoc debugging, and may unfortunately nudge people to use insecure connections. From users’ perspective, they just want to inspect an gRPC application in one console with the tool in another console.

The problem has been solved by many other softwares, like Grafana uses username and password, iPython notebook prints a generated auth token to the console. The admin server serves sensitive traffic statistics that should be protected. With easy-to-use credentials, we are promoting protected connections, and it will prevent gRPC from following the Envoy’s admin interface design.


## Implementation

TBD
