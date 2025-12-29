# A110: Child Channel Options

* Author(s): [Abhishek Agrawal](mailto:agrawalabhi@google.com)
* Approver: a11r
* Status: In Review
* Implemented in: Core, Java, Go
* Last updated: 2025-12-24
* Discussion at: [Mailing List Link TBD]

## Abstract

This proposal introduces a mechanism to configure "child channels"—channels created internally by gRPC components (such as xDS control plane channel). Currently, these channels are often opaque to the user, making it difficult to inject necessary configurations for metrics, tracing etc from the user application without relying on global state. This design proposes an approach for users to pass configuration options to these internal channels.

## Background

Complex gRPC ecosystems often require the creation of auxiliary channels that are not directly instantiated by the user application. Two primary examples are:

1. xDS (Extensible Discovery Service): When a user creates a channel with an xDS target, the gRPC library internally creates a separate channel to communicate with the xDS control plane.
2. External Authorization (ext_authz): As described in [gRFC A92](https://github.com/grpc/proposal/pull/481), the gRPC server or client may create an internal channel to contact an external authorization service.
3.  External Processing (ext_proc): As described in [gRFC A93](https://github.com/grpc/proposal/pull/484), filters may create internal channels to call external processing servers.

### Related Proposals

* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
* [A66: Otel Stats](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
* [A72: OpenTelemetry Tracing](https://github.com/grpc/proposal/blob/master/A72-open-telemetry-tracing.md)
* [A92: xDS ExtAuthz Support](https://github.com/grpc/proposal/pull/481)
* [A93: xDS ExtProc Support](https://github.com/grpc/proposal/pull/484)

### The Problem

The primary motivation for this feature is the need to configure observability on a per-channel basis and propagate them to child channels.

* StatsPlugins & Tracing: Users need to configure metric sinks (as described in gRFC A66 and A72) so that telemetry from internal channels is correctly tagged and exported.
* Interceptors: Users may need to apply specific interceptors (e.g., for authentication, logging, or tracing) to internal traffic.

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
* Conflict Resolution: Mandatory internal settings (`O_internal`) generally take precedence over user-provided child options (`O_child`) to ensure correctness. However, additive options (like Interceptors, StatsPlugins, or Metric Sinks) are combined.

### Shared Resources
Certain internal channels, specifically the **xDS Control Plane Client**, are often pooled and shared across multiple parent channels within a process based on the target URI (see [gRFC A27](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)).

If multiple Parent Channels (`P1`, `P2`) point to the same xDS target but provide *different* Child Channel Options (`O_child1`, `O_child2`):
* Behavior: The shared client is created using the options from the first parent channel that triggers its creation (e.g., `O_child1`).
* Subsequent Usage: When `P2` requests the client, it receives the existing shared client. `O_child2` is effectively ignored for that specific shared resource.

### Language Implementations

#### Java

In Java, the configuration will be achieved by accepting functions (callbacks). The API allows users to pass a `Consumer<ManagedChannelBuilder<?>>` (or a similar functional interface). When an internal library (e.g., xDS, gRPCLB) creates a child channel, it applies this user-provided function to the builder before building the channel.


* ##### Configuration Interface

  Use the standard `java.util.function.Consumer` and define a new public API interface, `ChildChannelConfigurer`, to encapsulate the configuration logic for auxiliary channels.

  ```java
  import java.util.function.Consumer;
  import io.grpc.ManagedChannelBuilder;

  // Captures the intent of the plugin.
  // Consumes a builder to modify it before the channel is built
  public interface ChildChannelConfigurer extends Consumer<ManagedChannelBuilder<?>> {
      // Inherits accept(T t) from Consumer
  }
  ``` 

* ##### API Changes

  * ManagedChannelBuilder: Add `ManagedChannelBuilder#childChannelConfigurer(ChildChannelConfigurer childChannelConfigurer)` to allow users to register this configurer, and `ManagedChannelBuilder#configureChannel(ManagedChannel parent)` to allow a new builder to inherit configuration from an existing parent channel.
  * ServerBuilder: Add `ServerBuilder#childChannelConfigurer(ChildChannelConfigurer configurer)` to allow users to provide configuration for any internal channels created by the server (e.g., connections to external authorization or processing services).


* ##### Internal Implementation

  The implementation propagates these configurations when creating internal channels. It leverages `configureChannel()` to act as a fusion point, automatically applying the `ChildChannelConfigurer` and other parent properties to the new builder. The implementation follows the pattern for global configurators and calls `.accept()` as soon as the builder is available.

* ##### Usage Example

  ```java
  // Define the configurer for internal child channels
  ChildChannelConfigurer myInternalConfig = (builder) -> {
      InternalManagedChannelBuilder.addMetricSink(builder, sink);
      InternalManagedChannelBuilder.interceptWithTarget(
          builder, openTelemetryMetricsModule::getClientInterceptor);
  };

  // Apply it to the parent channel
  ManagedChannel channel = ManagedChannelBuilder.forTarget("xds:///my-service")
      .childChannelConfigurer(myInternalConfig) // <--- Configuration injected here
      .build();    
  ```

* ##### Out-of-Band (OOB) Channels

  We do not propose applying child channel configurations to Out-of-Band (OOB) channels at this time. To maintain architectural flexibility and avoid breaking changes in the future, we will modify the implementation to use a `noOp()` MetricSink for OOB channels rather than inheriting the parent channel's sink.

  Furthermore, we acknowledge that certain configurations will not function out-of-the-box for `resolvingOobChannel` due to its specific initialization requirements.

#### Go

In Go, configuration for child channels is be achieved by passing a specific `DialOption` to the parent channel. This option encapsulates a slice of *other* `DialOption`s that are applied exclusively to any internal child channels created by the parent.

* ##### New API for Child Channel Options

  We introduce a new function in the grpc package that returns a `DialOption` specifically for child channels.

  ```go
  // WithChildChannelOptions returns a DialOption that specifies a list of options 
  // to be applied to any internal child channels (e.g., xDS control plane channels) 
  // created by this ClientConn.
  //
  // These options are NOT applied to the ClientConn returned by NewClient.
  func WithChildChannelOptions(opts ...DialOption) DialOption {
      return newFuncDialOption(func(o *dialOptions) {
          o.childChannelOptions = opts
      })
  }
  ```

* ##### Internal Mechanics

  A new field is added to the internal `dialOptions` struct to hold these options. When the xDS client (or any internal component) dials the control plane, it merges its own required options with the user-provided child options.

  ```go
  type dialOptions struct { 
       // ... existing fields ...
       // childChannelOptions holds options intended for internal child channels.
       childChannelOptions []DialOption
  }
  ```

* ##### Usage Example (User-Side Code)

  This design provides users with the flexibility to define independent configurations for parent and child channels within a single NewClient call. For example, a parent channel can be configured with transport security (mTLS) while the internal child channels (such as the xDS control plane connection) are configured with specific interceptors or a custom authority.

  ```go
  import (
      "google.golang.org/grpc"
      "google.golang.org/grpc/credentials/insecure"
  )

  func main() {
      // 1. Define configuration specifically for the internal control plane
      //    (e.g., a specific metrics interceptor or custom authority)
      internalOpts := []grpc.DialOption{
          grpc.WithUnaryInterceptor(MyMonitoringInterceptor),
          grpc.WithAuthority("xds-authority.example.com"),
      }

      // 2. Create the Parent Channel
      //    Pass the internal options using the WithChildChannelOptions wrapper.
      conn, err := grpc.NewClient("xds:///my-service",
          // Parent channel configuration
          grpc.WithTransportCredentials(insecure.NewCredentials()),
          
          // Child channel configuration (injected here)
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
  // A pointer argument key that internal components (like xDS) look for.
  // The value is a pointer to a grpc_channel_args struct containing the subset 
  // of options (e.g., specific socket mutators, user agents) for child channels.
  #define GRPC_ARG_CHILD_CHANNEL_ARGS "grpc.child_channel_args"
  ```

* ##### Internal Implementation

  Internal components that create channels (specifically `XdsClient`) are updated to look for this argument. When present, these arguments are merged with the default internal arguments required by the component.

* ##### Usage Example (User-Side Code)

  In this design, a user configures the parent channel by defining a subset of arguments intended for child channels, packing them into a standard `grpc_arg`, and passing them to the parent. This allows for granular control over internal components, such as the xDS control plane connection, without affecting the parent channel stack.

  ```c
  // 1. Prepare the Child Config (The Subset)
  // In this example, we configure a specific Socket Mutator for the child channel.
  grpc_socket_mutator* my_mutator = CreateMySocketMutator();
  grpc_arg child_arg = grpc_channel_arg_socket_mutator_create(my_mutator);
  grpc_channel_args child_args_struct = {1, &child_arg};

  // 2. Pack the Subset into the Parent's Arguments
  // The child_args_struct is wrapped as a pointer argument.
  // We use a VTable to ensure the nested struct is safely copied or
  // destroyed during the parent channel's lifetime.
  grpc_arg parent_arg = grpc_channel_arg_pointer_create(
      GRPC_ARG_CHILD_CHANNEL_ARGS, 
      &child_args_struct, 
      &grpc_channel_args_pointer_vtable
  );

  // 3. Create the Parent Channel
  // The parent channel receives the nested argument but does not apply
  // the socket mutator to itself.
  grpc_channel_args parent_args = {1, &parent_arg};
  auto channel = grpc::CreateCustomChannel(
      "xds:///my-service", 
      grpc::InsecureChannelCredentials(), 
      grpc::ChannelArguments::FromC(parent_args)
  );

  ```

## Rationale

### Why not Global Configuration?

We reject global configuration (static variables) because it prevents multi-tenant applications from isolating configurations. For example, one client may need to export metrics to Prometheus, while another in the same process exports to Cloud Monitoring.

### Why Language-Specific Approaches?

While the concept is cross-language, the mechanisms for channel creation differs significantly:

* Java relies heavily on the `Builder` pattern, making functional callbacks the most natural fit.
* Go uses functional options (`DialOption`), so passing a slice of options is idiomatic.
* Core uses a generic key-value map (`ChannelArgs`), maintaining consistency with the core C-core design.