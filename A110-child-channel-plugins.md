# A110: Child Channel Options

* Author(s): [Abhishek Agrawal](mailto:agrawalabhi@google.com)
* Approver: a11r
* Status: In Review
* Implemented in: Core, Java, Go
* Last updated: 2025-12-24
* Discussion at: https://groups.google.com/g/grpc-io/c/EBIp3uud-Bo

## Abstract

This proposal introduces a mechanism to configure "child channels", channels created internally by gRPC components (such as xDS control plane channel). Currently, these channels are often opaque to the user, making it difficult to inject necessary configurations for metrics, tracing etc from the user application. This design proposes an approach for users to pass configuration options to these internal channels.

## Background

Complex gRPC ecosystems often require the creation of auxiliary channels that are not directly instantiated by the 
user application. The primary examples are:

1. xDS (Extensible Discovery Service): When a user creates a channel with an xDS target, the gRPC library internally creates a separate channel to communicate with the xDS control plane.
2. External Authorization (ext_authz): As described in [gRFC A92](https://github.com/grpc/proposal/pull/481), the gRPC server or client may create an internal channel to contact an external authorization service.
3. External Processing (ext_proc): As described in [gRFC A93](https://github.com/grpc/proposal/pull/484), filters may 
   create internal channels to call external processing servers.

### Related Proposals

* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
* [A66: Otel Stats](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
* [A72: OpenTelemetry Tracing](https://github.com/grpc/proposal/blob/master/A72-open-telemetry-tracing.md)
* [A92: xDS ExtAuthz Support](https://github.com/grpc/proposal/pull/481)
* [A93: xDS ExtProc Support](https://github.com/grpc/proposal/pull/484)

### The Problem

The primary motivation for this feature is the need to configure observability on a per-child-channel basis.

* StatsPlugins & Tracing: Users need to configure metric sinks (as described in gRFC [A66](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md) and [A72](https://github.com/grpc/proposal/blob/master/A72-open-telemetry-tracing.md)) so that telemetry from internal channels is correctly tagged and exported.
* Interceptors: Users may need to apply specific interceptors (e.g., for logging, or tracing) to internal traffic.

These configurations cannot be set globally because different parts of an application may require different configurations, such as different metric backends or security credentials.

## Proposal

We introduce the concept of **Child Channel Options**. This is a configuration container attached to a parent channel that is strictly designated for use by its children. 

### Encapsulation
The user API must allow "nesting" of channel options. A user creating a Parent Channel `P` can provide a set of options `O_child`.
* `O_child` is opaque to `P`. `P` does not apply these options to itself.
* `O_child` is carried in `P`'s state, available for extraction by internal components.

### Propagation
When an internal component (e.g., an xDS client factory or an auth filter) attached to `P` needs to create a Child Channel `C`:
1.  It retrieves `O_child` from `P`.
2.  It applies `O_child` to the configuration of `C`.

### Precedence and Merging
The Child Channel `C` typically requires some internal configuration `O_internal` (e.g., specific target URIs, bootstrap credentials, or internal User-Agent strings).
* Merge Rule: `O_child` and `O_internal` are merged.
* Conflict Resolution: Mandatory internal settings (`O_internal`) generally take precedence over user-provided child options (`O_child`) to ensure correctness.

### Shared Resources
Certain internal channels, specifically the **xDS Control Plane Client**, are often pooled and shared across multiple parent channels within a process based on the target URI (see [gRFC A27](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)).

If multiple Parent Channels (`P1`, `P2`) point to the same xDS target but provide *different* Child Channel Options (`O_child1`, `O_child2`):
* Behavior: The shared client is created using the options from the first parent channel that triggers its creation (e.g., `O_child1`).
* Subsequent Usage: When `P2` requests the client, it receives the existing shared client. `O_child2` is effectively ignored for that specific shared resource.

### Language Implementations

#### Java

In Java, the configuration will be achieved by accepting functions (callbacks). The API allows users to pass a `Consumer<ManagedChannelBuilder<?>>` (or a similar functional interface). When an internal library (e.g., xDS, gRPCLB) creates a child channel, it applies this user-provided function to the builder before further configuring the channel.

* ##### Configuration Interface

  Use the standard `java.util.function.Consumer` and define a new public API interface, `ChildChannelConfigurer`, to encapsulate the configuration logic for auxiliary channels.

  ```java
  import java.util.function.Consumer;
  import io.grpc.ManagedChannelBuilder;

  // Captures the intent of the plugin.
  // Consumes a builder to modify it before the channel is built
  public interface ChannelConfigurer {
     /**
     * Configures the given channel builder.
     *
     * @param builder the channel builder to configure
     */
    void configure(ManagedChannelBuilder<?> builder);
  }
  ``` 

* ##### API Changes

  * ManagedChannelBuilder: Add `ManagedChannelBuilder#childChannelConfigurer(ChannelConfigurer channelConfigurer)` to 
    allow users to register this configurer.
  * XdsServerBuilder: Add `XdsServerBuilder#childChannelConfigurer(ChannelConfigurer configurer)` to allow users to 
    provide configuration for any internal channels created by the server (e.g., connections to external authorization or processing services).

* ##### Usage Example

  ```java
  // Define the configurer for internal child channels
  ChannelConfigurer myInternalConfig = (builder) -> {
      builder.addMetricSink(sink);
  };

  // Apply it to the parent channel
  ManagedChannel channel = ManagedChannelBuilder.forTarget("xds:///my-service")
      .childChannelConfigurer(myInternalConfig) // <--- Configuration injected here
      .build();    
  ```

#### Go

In Go, both the Client (`grpc.NewClient`) and the Server (`NewGRPCServer`) create internal child channels. We introduce mechanisms to pass `DialOption`s into these internal channels from both entry points.

* ##### New API for Child Channel Options

* Client-Side: `WithChildChannelOptions`

    For standard clients, we introduce a `DialOption` wrapper.

    ```go
    // WithChildChannelOptions returns a DialOption that specifies a list of 
    // DialOptions to be applied to any internal child channels.
    func WithChildChannelOptions(opts ...DialOption) DialOption {
        return newFuncDialOption(func(o *dialOptions) {
            o.childChannelOptions = opts
        })
    }
    ```

* Server-Side: `WithChildDialOptions`

    For xDS-enabled servers, we introduce a `ServerOption` wrapper. Since `xds.NewGRPCServer` creates an internal xDS client to fetch listener configurations, it requires a way to apply `DialOptions` (such as **Socket Options** or **Stats Handlers**) to that internal connection.

    ```go
    // WithChildDialOptions returns a ServerOption that specifies a list of 
    // DialOptions to be applied to the server's internal child channels 
    // (e.g., the xDS control plane connection).
    func WithChildDialOptions(opts ...DialOption) ServerOption {
        return newFuncServerOption(func(o *serverOptions) {
            o.childDialOptions = opts
        })
    }
    ```

* ##### Usage Example (User-Side Code)

  This design provides users with the flexibility to define independent configurations for parent and child channels within a single NewClient call. For example, a parent channel can be configured with transport security (mTLS) while the internal child channels (such as the xDS control plane connection) are configured with specific interceptors or a custom authority.

  ```go
  func main() {
      // Define configuration specifically for the internal control plane
      internalOpts := []grpc.DialOption{
          // Inject the OTel handler here. It will only measure traffic on the
          // internal child channels (e.g., to the xDS server).
          grpc.WithStatsHandler(otelHandler)
      }

      // Create the Parent Channel
      conn, err := grpc.NewClient("xds:///my-service",
          // Parent channel configuration (Data Plane)
          grpc.WithTransportCredentials(insecure.NewCredentials()),
          
          // Child channel configuration (Control Plane)
          // The OTel handler inside here applies ONLY to the child channels.
          grpc.WithChildChannelOptions(internalOpts...), 
      )
      
      if err != nil {
          log.Fatalf("failed to create client: %v", err)
      }
      defer conn.Close()
    
      // ... use conn ...
  }
  ```

##### Core (C/C++)

In gRPC Core, we utilize the existing `ChannelArgs` mechanism recursively to pass configuration to internal channels. We define a standard argument key whose value is a pointer to another `grpc_channel_args` structure. This "Nested Arguments" pattern allows the parent channel to carry a specific subset of arguments intended solely for its children.

* ##### Configuration Mechanism

  We define a new channel argument key. The value associated with this key is a pointer to a `grpc_channel_args` struct, managed via a pointer vtable to ensure correct ownership and copying.

  ```c
  // A pointer argument key. The value is a pointer to a grpc_channel_args 
  // struct containing the subset of options for child channels.
  #define GRPC_ARG_CHILD_CHANNEL_ARGS "grpc.child_channel.args"
  ```

* **API Changes**

  We add a helper method to the C++ `ChannelArguments` class to simplify packing the nested arguments safely.

  ```cpp
    // Sets the channel arguments to be used for child channels.
    void SetChildChannelArgs(const ChannelArguments& args);
  ```

* ##### Usage Example (User-Side Code)

  ```c
  // TODO(AgraVator): add example for configuring child channel observability
  ```

## Rationale

### Why not Global Configuration?

We reject global configuration (static variables) because it prevents multi-tenant applications from isolating configurations. For example, one client may need to export metrics to Prometheus, while another in the same process exports to Cloud Monitoring.

## Implementation

@AgraVator will be implementing immediately in grpc-java. A draft is available in
[PR 12578](https://github.com/grpc/grpc-java/pull/12578). Other languages will
follow, with work starting potentially in a few weeks.
