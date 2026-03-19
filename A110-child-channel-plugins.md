A110: Child Channel Options
----

* Author(s): [Abhishek Agrawal](mailto:agrawalabhi@google.com)
* Approver: @markdroth, @ejona86, @dfawley
* Status: In Review
* Implemented in: <language, ...>
* Last updated: 2026-03-18
* Discussion at: https://groups.google.com/g/grpc-io/c/EBIp3uud-Bo

## Abstract

There are several use cases where gRPC internally creates a "child channel".
Because these channels are created internally rather than being created by the
application, it is currently difficult to inject necessary configuration for
these channels, which makes it hard to configure things like metrics or tracing
from the application. This design proposes a mechanism to allow applications to
pass configuration options to these child channels.

## Background

Complex gRPC ecosystems often require the creation of auxiliary channels that
are not directly instantiated by the user application. The primary examples are:

1. xDS: When a user creates a channel with an xDS target, the gRPC library
   internally creates a separate channel to communicate with the xDS control
   plane.
2. External Authorization (ext_authz): As described
   in [gRFC A92](https://github.com/grpc/proposal/pull/481), the gRPC server or
   client may create an internal channel to contact an external authorization
   service.
3. External Processing (ext_proc): As described
   in [gRFC A93](https://github.com/grpc/proposal/pull/484), filters may create
   internal channels to call external processing servers.

The primary motivation for this feature is the need to configure observability
on a per-child-channel basis.

* StatsPlugins: Users use these plugins to configure metrics and tracing (as
  described in
  gRFC [A66](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
  and [A72](https://github.com/grpc/proposal/blob/master/A72-open-telemetry-tracing.md))
  so that telemetry from internal channels is correctly tagged and exported.
* Interceptors: Users may need to apply specific interceptors (e.g., for
  logging, or tracing) to internal traffic.

Global configuration is not sufficient for these use cases for the reasons
described in the 'Rationale' section below.

### Related Proposals

* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
* [A66: Otel Stats](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
* [A72: OpenTelemetry Tracing](https://github.com/grpc/proposal/blob/master/A72-open-telemetry-tracing.md)
* [A92: xDS ExtAuthz Support](https://github.com/grpc/proposal/pull/481)
* [A93: xDS ExtProc Support](https://github.com/grpc/proposal/pull/484)

## Proposal

We introduce the concept of **Child Channel Options**. This is a configuration
container attached to a parent channel/server that is strictly designated for
use by its children.

The user API must allow "nesting" of channel options. A user creating a Parent
Channel/Server `P` can provide a set of options `O_child`.

* `O_child` is opaque to `P`. `P` does not apply these options to itself.
* `O_child` is carried in `P`'s state, available for extraction by internal
  components.
* The configuration provided by `O_child` is strictly uniform across all child
  channels of a particular parent channel/server.

When an internal component (e.g., an xDS client factory) attached to `P`
needs to create a Child Channel `C`:

1. It retrieves `O_child` from `P`.
2. It applies `O_child` to the configuration of `C`.
3. It should also configure channel `C` to use `O_child` for its children.

The Child Channel `C` typically requires some internal
configuration `O_internal` (e.g., target URIs, or internal interceptors).

* Merge Rule: `O_child` and `O_internal` are merged. If the environment supports
  global channel options, `O_child` options override global channel options.
* Conflict Resolution: Mandatory internal settings (`O_internal`) generally take
  precedence over user-provided child options (`O_child`) to ensure correctness.

In some cases, child channels may be shared across multiple parent
channels/servers. For example, the xDS control plane channel is shared across
multiple channels or servers as described
in [gRFC A27](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).
However, it is possible for each parent channel or server to be created with
different child options. In such cases, we will do the following:

Consider an example where Parent Channels/Servers (`P1`, `P2`) point to the
same target but provide *different* Child Channel
Options (`O_child1`, `O_child2`):

* Behavior: The shared client is created using the options from the first parent
  channel or server that triggers its creation (e.g., `O_child1`).
* Subsequent Usage: When `P2` requests the client, it receives the existing
  shared client. `O_child2` is effectively ignored for that specific shared
  resource.

### Language Implementations

#### Java

In Java, the configuration will be achieved by accepting functions.
The API allows users to pass a `ManagedChannelBuilder<?> builder` or
`ServerBuilder<?> builder` (or a similar functional interface). When an internal
library (e.g., xDS, gRPCLB) creates a child channel, it applies this
user-provided function to the builder before further configuring the channel.

* ##### Configuration Interface

  Define a new public API interface, `ChannelConfigurer`, to encapsulate the
  configuration logic for channels.

  ```java

  import io.grpc.ManagedChannelBuilder;
  import io.grpc.ServerBuilder;

  // Captures the intent of the plugin.
  // Consumes a builder to modify it before further configuring the channel or server
  public interface ChannelConfigurer {
     /**
     * Configures the given channel builder.
     *
     * @param builder the channel builder to configure
     */
    default void configureChannelBuilder(ManagedChannelBuilder<?> builder) {}

     /**
     * Configures the given server builder.
     *
     * @param builder the server builder to configure
     */
    default void configureServerBuilder(ServerBuilder<?> builder) {}
  }
  ``` 

* ##### API Changes

    * ManagedChannelBuilder:
      Add `ManagedChannelBuilder#childChannelConfigurer(ChannelConfigurer channelConfigurer)`
      to allow users to register this configurer.
    * XdsServerBuilder:
      Add `XdsServerBuilder#childChannelConfigurer(ChannelConfigurer configurer)`
      to allow users to provide configuration for any internal channels created
      by the server (e.g., connections to external authorization or processing
      services).

* ##### Usage Example

  ```java
  // Define the configurer for internal child channels
  ChannelConfigurer myInternalConfig = new ChannelConfigurer() {
      @Override
      public void configureChannelBuilder(ManagedChannelBuilder<?> builder) {
          builder.addMetricSink(sink);
      }
  };

  // Apply it to the parent channel
  ManagedChannel channel = ManagedChannelBuilder.forTarget("xds:///my-service")
      .childChannelConfigurer(myInternalConfig) // <--- Configuration injected here
      .build();    
  ```

#### Go

In Go, both the Client (`grpc.NewClient`) and the Server (`NewGRPCServer`)
create internal child channels. We introduce mechanisms to pass `DialOption`s
into these internal channels from both entry points.

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

  For xDS-enabled servers, we introduce a `ServerOption` wrapper.
  Since `xds.NewGRPCServer` creates an internal xDS client to fetch listener
  configurations, it requires a way to apply `DialOptions` (such as **Socket
  Options** or **Stats Handlers**) to that internal connection.

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

  This design provides users with the flexibility to define independent
  configurations for parent and child channels within a single NewClient call.
  For example, a parent channel can be configured with transport security (mTLS)
  while the internal child channels (such as the xDS control plane connection)
  are configured with specific interceptors or a custom authority.

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

#### Core (C/C++)

In gRPC Core, we utilize the existing `ChannelArgs` mechanism recursively to
pass configuration to internal channels. We define a standard argument key whose
value is a pointer to another `grpc_channel_args` structure. This "Nested
Arguments" pattern allows the parent channel or server to carry a specific
subset of arguments intended solely for its children.

* ##### Configuration Mechanism

  We define a new channel argument key. The value associated with this key is a
  pointer to a `grpc_channel_args` struct, managed via a pointer vtable to
  ensure correct ownership and copying.

  ```c
  // A pointer argument key. The value is a pointer to a grpc_channel_args 
  // struct containing the subset of options for child channels.
  #define GRPC_ARG_CHILD_CHANNEL_ARGS "grpc.child_channel.args"
  ```

* ##### API Changes

  We add a helper method to the C++ `ChannelArguments` and `ServerBuilder`
  classes to simplify packing the nested arguments safely.

  ```cpp
    // Sets the channel arguments to be used for child channels.
    void SetChildChannelArgs(const ChannelArguments& args);
  ```

* ##### Usage Example (User-Side Code)

  An example of how this will work on the channel side:

  ```cpp
  // Create OTel StatsPlugin.
  grpc::OpenTelemetryPluginBuilder ot_plugin_builder;
  // ...set options on builder...
  auto ot_plugin = ot_plugin_builder.Build();
  assert(ot_plugin.ok());
  // Add the StatsPlugin to both child args and parent args.
  grpc::ChannelArguments child_args;
  grpc::ChannelArguments parent_args;
  ot_plugin->AddToChannelArguments(&child_args);
  ot_plugin->AddToChannelArguments(&parent_args);
  // Add child args to parent args.
  parent_args.SetChildChannelArgs(child_args);
  // Create channel with parent args.
  auto channel = grpc::CreateCustomChannel(
      "xds:///my-service", credentials, parent_args);
  ```

  An example of how this will work on the server side:

  ```cpp
  grpc::ServerBuilder server_builder;
  // Create OTel StatsPlugin.
  grpc::OpenTelemetryPluginBuilder ot_plugin_builder;
  // ...set options on builder...
  auto ot_plugin = ot_plugin_builder.Build();
  assert(ot_plugin.ok());
  // Add the StatsPlugin to both child args and the server builder.
  grpc::ChannelArguments child_args;
  ot_plugin->AddToChannelArguments(&child_args);
  ot_plugin->AddToServerBuilder(&server_builder);
  // Add the child args to the server builder.
  server_builder.SetChildChannelArgs(child_args);
  // Start the server.
  auto server = server_builder.BuildAndStart();
  ```

## Rationale

### Why not Global Configuration?

The primary use-case we care about is setting a `StatsPlugin` for one particular
channel, in which case we want that same `StatsPlugin` to also be used for any
child of that channel.

For example, let's say that we create two channels, one to target A and one to
target B, both of which create their own child channel to an `ext_authz` server
target Z. If we create the channel to target A with a specific `StatsPlugin`,
then we want that `StatsPlugin` to also be used for the child channel to target
Z created by the channel to target A. We do *not* want it to be used for the
child channel to target Z created by the channel to target B, because we did not
configure the `StatsPlugin` for the channel to target B.

We cannot achieve this using the global registry, for a couple of reasons.
First, the global registry can only select channels based on parameters like the
target URI. To attach our `StatsPlugin` to the internal target Z, we would have to
select it based on target Z, which would erroneously attach it to the child
channels for both A and B. Second, it is hard for the application that registers
the global `StatsPlugin` to know what target URIs will be used for internal
child channels.
