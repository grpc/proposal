# **gRFC AXX: Child Channel Configuration Plugins**

* **Author(s)**: [Abhishek Agrawal](mailto:agrawalabhi@google.com)
* **Status**: Draft
* **To be implemented in**: Core, Java, Go
* **Last updated**: 2025-12-23

## **Abstract**

This proposal introduces a mechanism to configure "child channels"—channels created internally by gRPC components (such as xDS control plane channel). Currently, these internal channels cannot easily inherit configuration (like metric sinks and interceptors) from the user application without relying on global state. This design proposes a language-specific approach to injecting configuration: using `functional interfaces` in Java, `DialOptions` in Go, and `ChannelArgs` in Core.

## **Background**

Complex gRPC ecosystems often require the creation of auxiliary channels that are not directly instantiated by the user application. Two primary examples are:

1. **xDS (Extensible Discovery Service)**: When a user creates a channel with an xDS target, the gRPC library internally creates a separate channel to communicate with the xDS control plane. Currently, users have limited ability to configure this internal control plane channel.
2. **Advanced Load Balancing (RLS, GrpcLB):** Policies like RLS (Route Lookup Service) and GrpcLB, as well as other high-level libraries built on top of gRPC, frequently create internal channels to communicate with look-aside load balancers or backends.

### **The Problem**

There is currently no standardized way to configure behavior for these child channels.

* **Metrics**: Users need to configure metric sinks so that telemetry from internal channels can be read and exported.
* **Interceptors**: Users may need to apply specific interceptors (e.g., for authentication, logging, or tracing) to internal traffic.
* **No Global State**: These configurations cannot be set globally (e.g., using static singletons) because different parts of an application may require different configurations, such as different metric backends or security credentials.

## **Proposal**

The proposal creates a "plugin" or configuration injection style for internal channels. The implementation varies by language to match existing idioms, but the goal remains consistent: allow the user to pass a configuration object or function that the internal channel factory applies during creation.

### **Java**

In Java, the configuration will be achieved by accepting functions (callbacks). The API allows users to pass a `Consumer<ManagedChannelBuilder<?>>` (or a similar functional interface). When an internal library (e.g., xDS, RLS, gRPCLB) creates a child channel, it applies this user-provided function to the builder before building the channel.


* #### 1\. Configuration Interface

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

* #### 2\. API Changes

  Add `ManagedChannelBuilder#childChannelConfigurer()` to allow users to register this configurer, and `ManagedChannelBuilder#configureChannel(ManagedChannel parent)` to allow a new builder to inherit configuration (including the `ChildChannelConfigurer` and `MetricRecorder`) from an existing parent channel.

* #### 3\. Internal Implementation

  The implementation propagates these configurations when creating internal channels. It leverages `configureChannel()` to act as a fusion point, automatically applying the `ChildChannelConfigurer` and other parent properties to the new builder. The implementation follows the pattern for global configurators and calls `.accept()` as soon as the builder is available.

* #### 4\. Usage Example

  ```java
  // 1. Define the configurer for internal child channels
  ChildChannelConfigurer myInternalConfig = (builder) -> {
    // Apply interceptors or configuration to the child channel builder
    builder.intercept(new MyAuthInterceptor());
    builder.maxInboundMessageSize(1024 * 1024);
  };

  // 2. Apply it to the parent channel
  ManagedChannel channel = ManagedChannelBuilder.forTarget("xds:///my-service")
    .childChannelConfigurer(myInternalConfig) // <--- Configuration injected here
    .build();
  ```

* #### 5\. Out-of-Band (OOB) Channels

  We do not propose applying child channel configurations to Out-of-Band (OOB) channels at this time. To maintain architectural flexibility and avoid breaking changes in the future, we will modify the implementation to use a `noOp()` MetricSink for OOB channels rather than inheriting the parent channel's sink.

  Furthermore, we acknowledge that certain configurations will not function out-of-the-box for `resolvingOobChannel` due to its specific initialization requirements.

### **Go**

In Go, configuration for child channels is be achieved by passing a specific `DialOption` to the parent channel. This option encapsulates a slice of other a slice of *other* `DialOption`s that are applied exclusively to any internal child channels created by the parent.

* #### 1\. New API for Child Channel Options

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

* #### 2\. Internal Mechanics

  A new field is added to the internal `dialOptions` struct to hold these options. When the xDS client (or any internal component) dials the control plane, it merges its own required options with the user-provided child options.

  ```go
  type dialOptions struct { 
       // ... existing fields ...
       // childChannelOptions holds options intended for internal child channels.
       childChannelOptions []DialOption
  }
  ```

* #### 3\. Usage Example (User-Side Code)

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

### **Core (C/C++)**

In gRPC Core, we utilize the existing `ChannelArgs` mechanism recursively to pass configuration to internal channels. We define a standard argument key whose value is a pointer to another `grpc_channel_args` structure. This "Nested Arguments" pattern allows the parent channel to carry a specific subset of arguments intended solely for its children.

* #### 1\. Configuration Mechanism

  We define a new channel argument key. The value associated with this key is a pointer to a `grpc_channel_args` struct, managed via a pointer vtable to ensure correct ownership and copying.

  ```c
  // A pointer argument key that internal components (like xDS) look for.
  // The value is a pointer to a grpc_channel_args struct containing the subset 
  // of options (e.g., specific socket mutators, user agents) for child channels.
  #define GRPC_ARG_CHILD_CHANNEL_ARGS "grpc.internal.child_channel_args"
  ```

* #### 2\. Internal Implementation

  Internal components that create channels (specifically `XdsClient`) are updated to look for this argument. When present, these arguments are merged with the default internal arguments required by the component.

* #### 3\. Usage Example (User-Side Code)

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

## **Rationale**

### **Why not Global Configuration?**

We reject global configuration (static variables) because it prevents multi-tenant applications from isolating configurations. For example, one client may need to export metrics to Prometheus, while another in the same process exports to Cloud Monitoring.

### **Why Language-Specific Approaches?**

While the concept is cross-language, the mechanisms for channel creation differs significantly:

* **Java** relies heavily on the `Builder` pattern, making functional callbacks the most natural fit.
* **Go** uses functional options (`DialOption`), so passing a slice of options is idiomatic.
* **Core** uses a generic key-value map (`ChannelArgs`), maintaining consistency with the core C-core design.