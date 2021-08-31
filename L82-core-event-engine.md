gRPC Core EventEngine API
----
* Author: AJ Heller (@drfloob)
* Approver: Mark Roth (@markdroth)
* Status: In Review
* Implemented in: gRPC Core
* Last updated: 2021.07.30
* Discussion at: https://groups.google.com/g/grpc-io/c/EamJ_ae_db0

## Abstract

This work replaces gRPC Core's iomgr with a public interface for custom,
pluggable implementations that we're calling EventEngines. EventEngines are
tasked with providing all I/O, task execution, and DNS resolution functionality
in a platform-agnostic way for gRPC Core and its wrapped languages.

Unlike the current polling model where threads are borrowed from the
application, EventEngines will bring their own polling and callback execution
threads. This public API will make it easier to integrate gRPC into external
event loops, it will support siloing events between gRPC channels and servers,
and it will provide better performance for the C++ Callback API.

## Background

gRPC Core and its wrapped languages rely internally on an event management
framework called iomgr. A small collection of homegrown, platform-specific
implementations exist, allowing I/O functionality to be customized to the
various platforms that gRPC supports (IOCP is used on Windows, epoll on Linux,
poll on Mac, etc).

Over the past few years, there have been multiple projects aimed at improving or
replacing iomgr. The EventEngine effort takes a slightly different approach to
things, but otherwise shares the same rationale as the previous effort.

### Related Proposals:

* This work supersedes [L69: EventManger API](https://github.com/grpc/proposal/pull/182)
* Supports [L67: C++ callback-based asynchronous API](https://github.com/grpc/proposal/pull/180) as a more performant solution than provided by the [CallbackAlternativeCQ](https://github.com/grpc/grpc/pull/25169).

## Proposal

### Concepts

An EventEngine is roughly composed of:

* A task runner which asynchronously executes callables with optional
  scheduling.
* An asynchronous DNS resolver.
* Network listener logic that accepts incoming connection requests, and
  performs network-server-related I/O operations.
* Client logic, support for making outgoing network connections.

### Common Data Types

```cpp
namespace grpc_event_engine {
namespace experimental {

/// Collection of parameters used to configure client and server endpoints. The
/// \a EndpointConfig maps string-valued keys to values of type int,
/// string_view, or void pointer. Each EventEngine implementation should
/// document its set of supported configuration options.
class EndpointConfig {
 public:
  virtual ~EndpointConfig() = default;
  using Setting = absl::variant<absl::monostate, int, absl::string_view, void*>;
  /// Returns the Setting for a specified key, or \a absl::monostate if there is
  /// no such entry. Caller does not take ownership of the resulting value.
  virtual Setting Get(absl::string_view key) const = 0;
};
```

The `SliceAllocator` provides memory buffers for EventEngine read/write
operations, allowing EventEngines to seamlessly participate in the
C++ `ResourceQuota` system.

```cpp
class SliceAllocator {
 public:
  virtual ~SliceAllocator() = default;
  using AllocateCallback = std::function<void(absl::Status)>;
  /// Requests \a size bytes from gRPC, and populates \a dest with the allocated
  /// slices. Ownership of the \a SliceBuffer is not transferred.
  ///
  /// gRPC provides a ResourceQuota system to cap the amount of memory used by
  /// the library. When a memory limit has been reached, slice allocation is
  /// interrupted to attempt to reclaim memory from participating gRPC
  /// internals. When there is sufficient memory available, slice allocation
  /// proceeds as normal.
  virtual absl::Status Allocate(size_t size, SliceBuffer* dest,
                        SliceAllocator::AllocateCallback cb) = 0;
};
```

The `SliceAllocatorFactory` allows `EventEnigine::Listener`s to create and
provide `SliceAllocator`s to `EventEngine::Endpoint`s as incoming connections
are accepted.

```cpp
class SliceAllocatorFactory {
 public:
  virtual ~SliceAllocatorFactory() = default;
  /// On Endpoint creation, call \a CreateSliceAllocator with the name of the
  /// endpoint peer (a URI string, most likely).
  virtual std::unique_ptr<SliceAllocator> CreateSliceAllocator(
      absl::string_view peer_name) = 0;
};
```

Finally, many `EventEngine` pieces rely on `Callback`s, `TaskHandle`s, and
`ResolvedAddress`es.

```cpp
class EventEngine {
 public:
  /// Basic callable function. The first argument to all callbacks is an
  /// absl::Status indicating the status of the operation associated with this
  /// callback. Each EventEngine method that takes a callback parameter, defines
  /// the expected sets and meanings of statuses for that use case.
  using Callback = std::function<void(absl::Status)>;
  /// Callback handle, used to cancel a callback.
  struct TaskHandle {
    intptr_t keys[2];
  };
  /// Thin wrapper around a platform-specific sockaddr type. A sockaddr struct
  /// exists on all platforms that gRPC supports.
  ///
  /// Platforms are expected to provide definitions for:
  /// * sockaddr
  /// * sockaddr_in
  /// * sockaddr_in6
  class ResolvedAddress {
   public:
    static constexpr socklen_t MAX_SIZE_BYTES = 128;

    ResolvedAddress(const sockaddr* address, socklen_t size);
    ResolvedAddress() = default;
    ResolvedAddress(const ResolvedAddress&) = default;
    const struct sockaddr* address() const;
    socklen_t size() const;

   private:
    char address_[MAX_SIZE_BYTES];
    socklen_t size_ = 0;
  };
  virtual ~EventEngine() = default;
```

### Endpoints

```cpp
  /// One end of a connection between a gRPC client and server. Endpoints are
  /// created when connections are established, and Endpoint operations are
  /// gRPC's primary means of communication.
  ///
  /// Endpoints must use the provided SliceAllocator for all data buffer memory
  /// allocations. gRPC allows applications to set memory constraints per
  /// Channel or Server, and the implementation depends on all dynamic memory
  /// allocation being handled by the quota system.
  class Endpoint {
   public:
    /// Shuts down all connections and invokes all pending read or write
    /// callbacks with an error status.
    virtual ~Endpoint() = default;
    /// Reads data from the Endpoint.
    ///
    /// When data is available on the connection, that data is moved into the
    /// \a buffer, and the \a on_read callback is called. The caller must ensure
    /// that the callback has access to the buffer when executed later.
    /// Ownership of the buffer is not transferred. Valid slices *may* be placed
    /// into the buffer even if the callback is invoked with a non-OK Status.
    ///
    /// There can be at most one outstanding read per Endpoint at any given
    /// time. An outstanding read is one in which the \a on_read callback has
    /// not yet been executed for some previous call to \a Read.  If an attempt
    /// is made to call \a Read while a previous read is still outstanding, the
    /// \a EventEngine must abort.
    ///
    /// For failed read operations, implementations should pass the appropriate
    /// statuses to \a on_read. For example, callbacks might expect to receive
    /// CANCELLED on endpoint shutdown.
    virtual void Read(Callback on_read, SliceBuffer* buffer) = 0;
    /// Writes data out on the connection.
    ///
    /// \a on_writable is called when the connection is ready for more data. The
    /// Slices within the \a data buffer may be mutated at will by the Endpoint
    /// until \a on_writable is called. The \a data in SliceBuffer will remain
    /// valid after calling \a Write, but its state is otherwise undefined.  All
    /// bytes in \a data must have been written before calling \a on_writable
    /// unless an error has occurred.
    ///
    /// There can be at most one outstanding write per Endpoint at any given
    /// time. An outstanding write is one in which the \a on_writable callback
    /// has not yet been executed for some previous call to \a Write.  If an
    /// attempt is made to call \a Write while a previous write is still
    /// outstanding, the \a EventEngine must abort.
    ///
    /// For failed write operations, implementations should pass the appropriate
    /// statuses to \a on_writable. For example, callbacks might expect to
    /// receive CANCELLED on endpoint shutdown.
    virtual void Write(Callback on_writable, SliceBuffer* data) = 0;
    /// Returns an address in the format described in DNSResolver. The returned
    /// values are expected to remain valid for the life of the Endpoint.
    virtual const ResolvedAddress& GetPeerAddress() const = 0;
    virtual const ResolvedAddress& GetLocalAddress() const = 0;
  };
```

### Listeners

```cpp
  /// Called when a new connection is established.
  ///
  /// If the connection attempt was not successful, implementations should pass
  /// the appropriate statuses to this callback. For example, callbacks might
  /// expect to receive DEADLINE_EXCEEDED statuses when appropriate, or
  /// CANCELLED statuses on EventEngine shutdown.
  using OnConnectCallback =
      std::function<void(absl::StatusOr<std::unique_ptr<Endpoint>>)>;

  /// Listens for incoming connection requests from gRPC clients and initiates
  /// request processing once connections are established.
  class Listener {
   public:
    /// Called when the listener has accepted a new client connection.
    using AcceptCallback = std::function<void(
        std::unique_ptr<Endpoint>, const SliceAllocator& slice_allocator)>;
    virtual ~Listener() = default;
    /// Bind an address/port to this Listener.
    ///
    /// It is expected that multiple addresses/ports can be bound to this
    /// Listener before Listener::Start has been called. Returns either the
    /// bound port or an appropriate error status.
    virtual absl::StatusOr<int> Bind(const ResolvedAddress& addr) = 0;
    virtual absl::Status Start() = 0;
  };
```

### Creating Listeners and Client Connection

```cpp
  /// Factory method to create a network listener / server.
  ///
  /// Once a \a Listener is created and started, the \a on_accept callback will
  /// be called once asynchronously for each established connection. Note that
  /// unlike other callbacks, there is no status code parameter since the
  /// callback will only be called in healthy scenarios where connections can be
  /// accepted.
  ///
  /// This method may return a non-OK status immediately if an error was
  /// encountered in any synchronous steps required to create the Listener.
  ///
  /// The provided \a SliceAllocatorFactory is used to create \a SliceAllocators
  /// for Endpoint construction.
  virtual absl::StatusOr<std::unique_ptr<Listener>> CreateListener(
      Listener::AcceptCallback on_accept, const EndpointConfig& args,
      std::unique_ptr<SliceAllocatorFactory> slice_allocator_factory) = 0;
  /// Creates a client network connection to a remote network listener.
  ///
  /// May return an error status immediately if there was a failure in the
  /// synchronous part of establishing a connection. In that event, the \a
  /// on_connect callback *will not* have been executed. Otherwise, it is
  /// expected that the \a on_connect callback will be asynchronously executed
  /// exactly once by the EventEngine.
  ///
  /// Implementation Note: it is important that the \a slice_allocator be used
  /// for all read/write buffer allocations in the EventEngine implementation.
  /// This allows gRPC's \a ResourceQuota system to monitor and control memory
  /// usage with graceful degradation mechanisms. Please see the
  /// \a SliceAllocator API for more information.
  virtual absl::Status Connect(OnConnectCallback on_connect,
                               const ResolvedAddress& addr,
                               const EndpointConfig& args,
                               std::unique_ptr<SliceAllocator> slice_allocator,
                               absl::Time deadline) = 0;
```

### DNS Resolution

```cpp
  /// Provides asynchronous resolution.
  class DNSResolver {
   public:
    /// Task handle for DNS Resolution requests.
    struct LookupTaskHandle {
      intptr_t key[2];
    };
    /// DNS SRV record type.
    struct SRVRecord {
      std::string host;
      int port = 0;
      int priority = 0;
      int weight = 0;
    };
    /// Called with the collection of sockaddrs that were resolved from a given
    /// target address.
    using LookupHostnameCallback =
        std::function<void(absl::StatusOr<std::vector<ResolvedAddress>>)>;
    /// Called with a collection of SRV records.
    using LookupSRVCallback =
        std::function<void(absl::StatusOr<std::vector<SRVRecord>>)>;
    /// Called with the result of a TXT record lookup
    using LookupTXTCallback = std::function<void(absl::StatusOr<std::string>)>;

    virtual ~DNSResolver() = default;

    /// Asynchronously resolve an address.
    ///
    /// \a default_port may be a non-numeric named service port, and will only
    /// be used if \a address does not already contain a port component.
    ///
    /// When the lookup is complete, the \a on_resolve callback will be invoked
    /// with a status indicating the success or failure of the lookup.
    /// Implementations should pass the appropriate statuses to the callback.
    /// For example, callbacks might expect to receive DEADLINE_EXCEEDED when
    /// the deadline is exceeded or CANCELLED if the lookup was cancelled.
    virtual LookupTaskHandle LookupHostname(LookupHostnameCallback on_resolve,
                                            absl::string_view address,
                                            absl::string_view default_port,
                                            absl::Time deadline) = 0;
    /// Asynchronously perform an SRV record lookup.
    ///
    /// \a on_resolve has the same meaning and expectations as \a
    /// LookupHostname's \a on_resolve callback.
    virtual LookupTaskHandle LookupSRV(LookupSRVCallback on_resolve,
                                       absl::string_view name,
                                       absl::Time deadline) = 0;
    /// Asynchronously perform a TXT record lookup.
    ///
    /// \a on_resolve has the same meaning and expectations as \a
    /// LookupHostname's \a on_resolve callback.
    virtual LookupTaskHandle LookupTXT(LookupTXTCallback on_resolve,
                                       absl::string_view name,
                                       absl::Time deadline) = 0;
    /// Cancel an asynchronous lookup operation.
    virtual void TryCancelLookup(LookupTaskHandle handle) = 0;
  };
```

### Task Execution

```cpp
  /// Creates and returns an instance of a DNSResolver.
  virtual std::unique_ptr<DNSResolver> GetDNSResolver() = 0;

  /// Executes a callback as soon as possible.
  ///
  /// The \a fn callback's \a status argument is used to indicate whether it was
  /// executed normally. For example, the status may be CANCELLED if
  /// \a TryCancel was called, or if the EventEngine is being shut down.
  ///
  /// It is an error to call \a Run after the EventEngine is shut down.
  /// The provided \a fn will be executed with an error status immediately,
  /// possibly in the same call stack.
  virtual TaskHandle Run(Callback fn) = 0;
  /// Synonymous with scheduling an alarm to run at time \a when.
  ///
  /// The callback \a fn will execute when either when time \a when arrives
  /// (receiving status OK), or when the \a fn is cancelled (receiving status
  /// CANCELLED). The callback is guaranteed to be called exactly once.
  ///
  /// It is an error to call \a RunAt after the EventEngine is shut down.
  /// The provided \a fn will be executed with an error status immediately,
  /// possibly in the same call stack.
  virtual TaskHandle RunAt(absl::Time when, Callback fn) = 0;
  /// Attempts to cancel a callback.
  /// Note that this is a "best effort" cancellation. No guarantee is made that
  /// the callback will be cancelled, the call could be in any stage.
  ///
  /// There are three scenarios in which we may cancel a scheduled task:
  ///   1. We cancel the execution before it has run.
  ///   2. The callback has already run.
  ///   3. We can't cancel it because it is "in flight".
  ///
  /// In all cases, the cancellation is still considered successful, the
  /// callback will be run exactly once from either cancellation or from its
  /// activation.
  virtual void TryCancel(TaskHandle handle) = 0;
};

}  // namespace experimental
}  // namespace grpc_event_engine
```

### Application APIs for providing custom EventEngines

*Caveat: This is not final, feedback is welcome!*

There are three ways in which EventEngines can be provided to gRPC.

#### Setting an EventEngine instance for a Channel

EventEngines can be configured per channel, which allows customization for both
clients and servers. To set a custom EventEngine for a client channel, you can
do something like the following:

```cpp
ChannelArguments args;
std::shared_ptr<EventEngine> engine = std::make_shared<MyEngine>(...);
args.SetEventEngine(engine);
MyAppClient client(grpc::CreateCustomChannel(
    "localhost:50051", grpc::InsecureChannelCredentials(), args));
```

#### Setting an EventEngine instance for a Server

A gRPC server can use a custom EventEngine by calling the
`ServerBuilder::SetEventEngine` method:

```cpp
ServerBuilder builder;
std::shared_ptr<EventEngine> engine = std::make_shared<MyEngine>(...);
builder.SetEventEngine(engine);
std::unique_ptr<Server> server(builder.BuildAndStart());
server->Wait();
```

#### Providing a default EventEngine factory

TODO: this has not been thoroughly discussed.

Before gRPC is initialized, an application can specify an EventEngine Factory
method that will be used within gRPC when a channel- or server-specific
EventEngine is unavailable. gRPC needs to perform I/O, run tasks, and use timers
in scenarios that are not directly attached to channels and servers.  Providing
a custom EventEngine factory will allow your application to avoid the creation
of the default EventEngine provided by gRPC.

### Lifetime and Ownership

gRPC will take shared ownership of application-provided EventEngines via
std::shared\_ptrs to ensure that the engines remain available until they are no
longer needed. Depending on the use case, EventEngines may need to live until
gRPC is shut down. Any EventEngine that is provided to gRPC is expected to
remain valid for as long as gRPC holds a pointer to it. Application-provided
EventEngines will not be shut down automatically when gRPC is shut down, it is
the application's responsibility to shut down and clean up EventEngines when
they are no longer needed.

### Synchronous Server Implementation

Instead of the separate thread pool that gRPC creates for C++ synchronous
servers, the synchronous server implementation will use the same threads
provided by the EventEngine as will be used for the asynchronous API.

### Temporary protections

The EventEngine can be enabled at compile time by defining the C/C++ macro
`GRPC_USE_EVENT_ENGINE`. EventEngines will not be enabled by default on any
platform to begin with.

The EventEngine API will exist in the `grpc_event_engine::experimental`
namespace until it has stabilized. Stabilization will likely be signaled by
having all platforms supported sufficiently by the default EventEngine, and
having all other primary EventEngine use cases exercised without needing to make
further API changes.

The EventEngine rollout is planned to be on a per-platform basis. Once we
establish that the default EventEngine implementation is sufficient for a
specific platform, we will change the build to replace iomgr with EventEngine on
that platform.

## Limitations

### No immediate goal for wrapped languages to provide custom EventEngines

The ability to provide custom EventEngines at runtime will initially be
available in the C++ and C-core APIs only. Adding support for custom
EventEngines written in wrapped languages, or providing EventEngines at runtime
from wrapped languages, is not a focus of this work.  Challenges may include the
management of dll-barrier crossing issues, and potential performance losses.

### Public file descriptor operations are not supported for custom EventEngines

gRPC public APIs contain methods that revolve around the concept of a system
file descriptor (fd) for network operations. This is not a cross-platform
concept, and many of the platforms that gRPC runs on do not natively support
fds. For that reason, we did not want to include any mention of fds in the
EventEngine API.

Since we're committed to not making any breaking changes with this work, we have
made a compromise to continue to support those fd-specific methods in the
default EventEngine implementation alone. In other words, if your application
provides a custom EventEngine, the fd-specific APIs will fail.

A sampling of APIs that will only be supported with the default EventEngine
implementation:

* [CreateInsecureChannelFromFd](https://github.com/grpc/grpc/blob/v1.38.0/include/grpcpp/create_channel_posix.h#L32-L46)
* [AddInsecureChannelFromFd](https://github.com/grpc/grpc/blob/v1.38.0/include/grpcpp/server_posix.h#L31-L36)
* [SetSocketMutator](https://github.com/grpc/grpc/blob/v1.38.0/include/grpcpp/support/channel_arguments.h#L73-L80)
* [grpc\_insecure\_channel\_create\_from\_fd](https://github.com/grpc/grpc/blob/v1.38.0/include/grpc/grpc_posix.h#L38-L42)

## Rationale

The rationale is essentially unchanged from past efforts. Quoting
[L69](https://github.com/grpc/proposal/pull/182) directly, since the rationale
was already well-stated:

* The current iomgr APIs are a mess; the semantics are extremely confusing,
  which makes our code very hard to maintain and reason about. This complexity
  slows down our velocity on new features and makes it hard to debug and
  maintain existing code.
* The current iomgr APIs are extremely invasive and viral, requiring
  pollset\_sets to be plumbed into many different parts of the client\_channel
  code.
* There are known bugs in several of our common polling engines, and we don't
  really have anyone left on the team who understands them well enough to try to
  fix them.
* From a strategic point of view, it does not make sense for us to spend time
  and effort to maintain multiple polling APIs.

To address these problems, we would like to replace the iomgr APIs with the
EventEngine API, and provide a default EventEngine implementation that utilizes
third-party, cross-platform I/O libraries that work on the various platforms
that gRPC supports.

## The Team

The design team is lead by AJ Heller (@drfloob), and includes Mark Roth
(@markdroth) and Nicolas Noble (@nicolasnoble).

The default implementation work is split between AJ and Nico: AJ will write an
iomgr implementation that is driven by an EventEngine (using the EventEngine API
alone), and Nico will write an EventEngine prototype using libuv.

When it comes time to do so, Mark will remove the pollset\_set code, and AJ will
likely remove unneeded iomgr implementations.

Richard Belleville (@gnossen) has volunteered to write a custom gevent-based
EventEngine for python. Lidi Zheng (@lidizheng) will work on the asyncio-based
implementation.

## TODO

Yet to be defined:

* The specific changes required for all wrapped languages
* Plans to deprecate the gRPC-Core-internal completion queue model
