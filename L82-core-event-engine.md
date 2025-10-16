gRPC Core EventEngine API
----
* Author: AJ Heller (@drfloob)
* Approver: Mark Roth (@markdroth)
* Status: In Review
* Implemented in: gRPC Core
* Last updated: 2022.02.10
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

The `MemoryAllocator` provides memory for EventEngine read/write operations,
allowing EventEngines to seamlessly participate in the C++ `ResourceQuota`
system. Methods like `New` and `MakeUnique` provide easy means of object
construction, while `Reserve` and `Release` let EventEngines report bytes that
were allocated elsewhere.

```cpp
// Tracks memory allocated by one system.
// Is effectively a thin wrapper/smart pointer for a MemoryAllocatorImpl,
// providing a convenient and stable API.
class MemoryAllocator {
 public:
  /// Construct a MemoryAllocator given an internal::MemoryAllocatorImpl
  /// implementation. The constructed MemoryAllocator will call
  /// MemoryAllocatorImpl::Shutdown() upon destruction.
  explicit MemoryAllocator(
      std::shared_ptr<internal::MemoryAllocatorImpl> allocator)
      : allocator_(std::move(allocator)) {}
  // Construct an invalid MemoryAllocator.
  MemoryAllocator() : allocator_(nullptr) {}
  ~MemoryAllocator() {
    if (allocator_ != nullptr) allocator_->Shutdown();
  }

  MemoryAllocator(const MemoryAllocator&) = delete;
  MemoryAllocator& operator=(const MemoryAllocator&) = delete;

  MemoryAllocator(MemoryAllocator&&) = default;
  MemoryAllocator& operator=(MemoryAllocator&&) = default;

  /// Drop the underlying allocator and make this an empty object.
  /// The object will not be usable after this call unless it's a valid
  /// allocator is moved into it.
  void Reset() {
    if (allocator_ != nullptr) allocator_->Shutdown();
    allocator_.reset();
  }

  /// Reserve bytes from the quota.
  /// If we enter overcommit, reclamation will begin concurrently.
  /// Returns the number of bytes reserved.
  size_t Reserve(MemoryRequest request) { return allocator_->Reserve(request); }

  /// Release some bytes that were previously reserved.
  void Release(size_t n) { return allocator_->Release(n); }

  //
  // The remainder of this type are helper functions implemented in terms of
  // Reserve/Release.
  //

  /// An automatic releasing reservation of memory.
  class Reservation {
   public:
    Reservation() = default;
    Reservation(const Reservation&) = delete;
    Reservation& operator=(const Reservation&) = delete;
    Reservation(Reservation&&) = default;
    Reservation& operator=(Reservation&&) = default;
    ~Reservation() {
      if (allocator_ != nullptr) allocator_->Release(size_);
    }

   private:
    friend class MemoryAllocator;
    Reservation(std::shared_ptr<internal::MemoryAllocatorImpl> allocator,
                size_t size)
        : allocator_(std::move(allocator)), size_(size) {}

    std::shared_ptr<internal::MemoryAllocatorImpl> allocator_;
    size_t size_ = 0;
  };

  /// Reserve bytes from the quota and automatically release them when
  /// Reservation is destroyed.
  Reservation MakeReservation(MemoryRequest request) {
    return Reservation(allocator_, Reserve(request));
  }

  /// Allocate a new object of type T, with constructor arguments.
  /// The returned type is wrapped, and upon destruction the reserved memory
  /// will be released to the allocator automatically. As such, T must have a
  /// virtual destructor so we can insert the necessary hook.
  template <typename T, typename... Args>
  typename std::enable_if<std::has_virtual_destructor<T>::value, T*>::type New(
      Args&&... args) {
    // Wrap T such that when it's destroyed, we can release memory back to the
    // allocator.
    class Wrapper final : public T {
     public:
      explicit Wrapper(std::shared_ptr<internal::MemoryAllocatorImpl> allocator,
                       Args&&... args)
          : T(std::forward<Args>(args)...), allocator_(std::move(allocator)) {}
      ~Wrapper() override { allocator_->Release(sizeof(*this)); }

     private:
      const std::shared_ptr<internal::MemoryAllocatorImpl> allocator_;
    };
    Reserve(sizeof(Wrapper));
    return new Wrapper(allocator_, std::forward<Args>(args)...);
  }

  /// Construct a unique_ptr immediately.
  template <typename T, typename... Args>
  std::unique_ptr<T> MakeUnique(Args&&... args) {
    return std::unique_ptr<T>(New<T>(std::forward<Args>(args)...));
  }

  /// Allocate a slice, using MemoryRequest to size the number of returned
  /// bytes. For a variable length request, check the returned slice length to
  /// verify how much memory was allocated. Takes care of reserving memory for
  /// any relevant control structures also.
  grpc_slice MakeSlice(MemoryRequest request);

  /// A C++ allocator for containers of T.
  template <typename T>
  class Container {
   public:
    using value_type = T;

    /// Construct the allocator: \a underlying_allocator is borrowed, and must
    /// outlive this object.
    explicit Container(MemoryAllocator* underlying_allocator)
        : underlying_allocator_(underlying_allocator) {}
    template <typename U>
    explicit Container(const Container<U>& other)
        : underlying_allocator_(other.underlying_allocator()) {}

    MemoryAllocator* underlying_allocator() const {
      return underlying_allocator_;
    }

    T* allocate(size_t n) {
      underlying_allocator_->Reserve(n * sizeof(T));
      return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    void deallocate(T* p, size_t n) {
      ::operator delete(p);
      underlying_allocator_->Release(n * sizeof(T));
    }

   private:
    MemoryAllocator* underlying_allocator_;
  };

 protected:
  /// Return a pointer to the underlying implementation.
  /// Note that the interface of said implementation is unstable and likely to
  /// change at any time.
  internal::MemoryAllocatorImpl* get_internal_impl_ptr() {
    return allocator_.get();
  }

  const internal::MemoryAllocatorImpl* get_internal_impl_ptr() const {
    return allocator_.get();
  }

 private:
  std::shared_ptr<internal::MemoryAllocatorImpl> allocator_;
};

// Wrapper type around std::vector to make initialization against a
// MemoryAllocator based container allocator easy.
template <typename T>
class Vector : public std::vector<T, MemoryAllocator::Container<T>> {
 public:
  explicit Vector(MemoryAllocator* allocator)
      : std::vector<T, MemoryAllocator::Container<T>>(
            MemoryAllocator::Container<T>(allocator)) {}
};
```

The `MemoryAllocatorFactory` allows `EventEnigine::Listener`s to create and
provide `MemoryAllocator`s to `EventEngine::Endpoint`s as incoming connections
are accepted.

```cpp
class MemoryAllocatorFactory {
 public:
  virtual ~MemoryAllocatorFactory() = default;
  /// On Endpoint creation, call \a CreateMemoryAllocator to create a new
  /// allocator for the endpoint.
  /// \a name is used to label the memory allocator in debug logs.
  /// Typically we'll want to:
  ///    auto allocator = factory->CreateMemoryAllocator(peer_address_string);
  ///    auto* endpoint = allocator->New<MyEndpoint>(std::move(allocator), ...);
  virtual MemoryAllocator CreateMemoryAllocator(absl::string_view name) = 0;
};
```

The task execution metohds (`Run*`) support both `std::function` and a custom
`Closure` callback types. `Closure`s can be implemented more efficiently in some
circumstances, leading to performance improvements in performance-sensitive
code.

```cpp
  /// A custom closure type for EventEngine task execution.
  ///
  /// Throughout the EventEngine API, \a Closure ownership is retained by the
  /// caller - the EventEngine will never delete a Closure, and upon
  /// cancellation, the EventEngine will simply forget the Closure exists. The
  /// caller is responsible for all necessary cleanup.
  class Closure {
   public:
    Closure() = default;
    // Closure's are an interface, and thus non-copyable.
    Closure(const Closure&) = delete;
    Closure& operator=(const Closure&) = delete;
    // Polymorphic type => virtual destructor
    virtual ~Closure() = default;
    // Run the contained code.
    virtual void Run() = 0;
  };
```

Finally, a few common `EventEngine` odds-and-ends:

```cpp
class EventEngine {
 public:
  /// Represents a scheduled task.
  ///
  /// \a TaskHandles are returned by \a Run* methods, and can be given to the
  /// \a Cancel method.
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
  /// Endpoints must use the provided MemoryAllocator for all data buffer memory
  /// allocations. gRPC allows applications to set memory constraints per
  /// Channel or Server, and the implementation depends on all dynamic memory
  /// allocation being handled by the quota system.
  class Endpoint {
   public:
    /// Shuts down all connections and invokes all pending read or write
    /// callbacks with an error status.
    virtual ~Endpoint() = default;
    /// A struct representing optional arguments that may be provided to an
    /// EventEngine Endpoint Read API  call.
    ///
    /// Passed as argument to an Endpoint \a Read
    struct ReadArgs {
      // A suggestion to the endpoint implementation to read at-least the
      // specified number of bytes over the network connection before marking
      // the endpoint read operation as complete. gRPC may use this argument
      // to minimize the number of endpoint read API calls over the lifetime
      // of a connection.
      int64_t read_hint_bytes;
    };
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
    virtual void Read(std::function<void(absl::Status)> on_read,
                      SliceBuffer* buffer, const ReadArgs* args = nullptr) = 0;
    /// A struct representing optional arguments that may be provided to an
    /// EventEngine Endpoint Write API call.
    ///
    /// Passed as argument to an Endpoint \a Write
    struct WriteArgs {
      // Represents private information that may be passed by gRPC for
      // select endpoints expected to be used only within google.
      void* google_specific = nullptr;
      // A suggestion to the endpoint implementation to group data to be written
      // into frames of the specified max_frame_size. gRPC may use this
      // argument to dynamically control the max sizes of frames sent to a
      // receiver in response to high receiver memory pressure.
      int64_t max_frame_size;
    };
    /// Writes data out on the connection.
    ///
    /// \a on_writable is called when the connection is ready for more data. The
    /// Slices within the \a data buffer may be mutated at will by the Endpoint
    /// until \a on_writable is called. The \a data SliceBuffer will remain
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
    virtual void Write(std::function<void(absl::Status)> on_writable,
                       SliceBuffer* data, const WriteArgs* args = nullptr) = 0;
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
        std::unique_ptr<Endpoint>, MemoryAllocator memory_allocator)>;
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
  /// be called once asynchronously for each established connection. This method
  /// may return a non-OK status immediately if an error was encountered in any
  /// synchronous steps required to create the Listener. In this case,
  /// \a on_shutdown will never be called.
  ///
  /// If this method returns a Listener, then \a on_shutdown will be invoked
  /// exactly once, when the Listener is shut down. The status passed to it will
  /// indicate if there was a problem during shutdown.
  ///
  /// The provided \a MemoryAllocatorFactory is used to create \a
  /// MemoryAllocators for Endpoint construction.
  virtual absl::StatusOr<std::unique_ptr<Listener>> CreateListener(
      Listener::AcceptCallback on_accept,
      std::function<void(absl::Status)> on_shutdown,
      const EndpointConfig& config,
      std::unique_ptr<MemoryAllocatorFactory> memory_allocator_factory) = 0;
  /// Creates a client network connection to a remote network listener.
  ///
  /// Even in the event of an error, it is expected that the \a on_connect
  /// callback will be asynchronously executed exactly once by the EventEngine.
  /// A connection attempt can be cancelled using the \a CancelConnect method.
  ///
  /// Implementation Note: it is important that the \a memory_allocator be used
  /// for all read/write buffer allocations in the EventEngine implementation.
  /// This allows gRPC's \a ResourceQuota system to monitor and control memory
  /// usage with graceful degradation mechanisms. Please see the \a
  /// MemoryAllocator API for more information.
  virtual ConnectionHandle Connect(OnConnectCallback on_connect,
                                   const ResolvedAddress& addr,
                                   const EndpointConfig& args,
                                   MemoryAllocator memory_allocator,
                                   absl::Time deadline) = 0;

  /// Request cancellation of a connection attempt.
  ///
  /// If the associated connection has already been completed, it will not be
  /// cancelled, and this method will return false.
  ///
  /// If the associated connection has not been completed, it will be cancelled,
  /// and this method will return true. The \a OnConnectCallback will not be
  /// called.
  virtual bool CancelConnect(ConnectionHandle handle) = 0;
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
    /// For example, callbacks might expect to receive DEADLINE_EXCEEDED or
    /// NOT_FOUND.
    ///
    /// If cancelled, \a on_resolve will not be executed.
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
    ///
    /// This shares the same semantics with \a EventEngine::Cancel: successfully
    /// cancelled lookups will not have their callbacks executed, and this
    /// method returns true.
    virtual bool CancelLookup(LookupTaskHandle handle) = 0;
  };
  /// Creates and returns an instance of a DNSResolver.
  virtual std::unique_ptr<DNSResolver> GetDNSResolver() = 0;
```

### Task Execution

```cpp
  /// Asynchronously executes a task as soon as possible.
  ///
  /// \a Closures scheduled with \a Run cannot be cancelled. The \a closure will
  /// not be deleted after it has been run, ownership remains with the caller.
  virtual void Run(Closure* closure) = 0;
  /// Asynchronously executes a task as soon as possible.
  ///
  /// \a Closures scheduled with \a Run cannot be cancelled. Unlike the
  /// overloaded \a Closure alternative, the std::function version's \a closure
  /// will be deleted by the EventEngine after the closure has been run.
  ///
  /// This version of \a Run may be less performant than the \a Closure version
  /// in some scenarios. This overload is useful in situations where performance
  /// is not a critical concern.
  virtual void Run(std::function<void()> closure) = 0;
  /// Synonymous with scheduling an alarm to run at time \a when.
  ///
  /// The \a closure will execute when time \a when arrives unless it has been
  /// cancelled via the \a Cancel method. If cancelled, the closure will not be
  /// run, nor will it be deleted. Ownership remains with the caller.
  virtual TaskHandle RunAt(absl::Time when, Closure* closure) = 0;
  /// Synonymous with scheduling an alarm to run at time \a when.
  ///
  /// The \a closure will execute when time \a when arrives unless it has been
  /// cancelled via the \a Cancel method. If cancelled, the closure will not be
  /// run. Unilke the overloaded \a Closure alternative, the std::function
  /// version's \a closure will be deleted by the EventEngine after the closure
  /// has been run, or upon cancellation.
  ///
  /// This version of \a RunAt may be less performant than the \a Closure
  /// version in some scenarios. This overload is useful in situations where
  /// performance is not a critical concern.
  virtual TaskHandle RunAt(absl::Time when, std::function<void()> closure) = 0;
  /// Request cancellation of a task.
  ///
  /// If the associated closure has already been scheduled to run, it will not
  /// be cancelled, and this function will return false.
  ///
  /// If the associated callback has not been scheduled to run, it will be
  /// cancelled, and the associated std::function or \a Closure* will not be
  /// executed. In this case, Cancel will return true.
  virtual bool Cancel(TaskHandle handle) = 0;
};

}  // namespace experimental
}  // namespace grpc_event_engine
```

### Application APIs for providing custom EventEngines

*Caveat: This is not final, feedback is welcome!*

There will be multiple ways in which applications can provide custom
EventEngines to gRPC.

* An application can specify a _global EventEngine provider_ that returns
  EventEngine(s) for any client code that requires one.
* EventEngines can be specified _per Channel_ or _per Server_.

During the experimental period, details will be refined regarding EventEngine
lifetimes and ownership stories. In the initial phases, gRPC will rely on a
global EventEngine factory (which applications can provide), with the
expectation that EventEngines never need to be shut down.

### Synchronous Server Implementation

Instead of the separate thread pool that gRPC creates for C++ synchronous
servers, the synchronous server implementation will use the same threads
provided by the EventEngine as will be used for the asynchronous API.

### Temporary protections

The EventEngine API will exist in the `grpc_event_engine::experimental`
namespace until it has stabilized. Stabilization will likely be signaled by
having all platforms supported sufficiently by the default EventEngine, and
having all other primary EventEngine use cases exercised without needing to make
further API changes.

The EventEngine rollout is planned to happen on a feature-by-feature basis, and
one callsite at a time. In some scenarios, we'll maintain both EventEngine and
the legacy versions of code, with the option to select behavior on startup.
(Details to be documented)

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

## Rationale for this project

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

AJ Heller (@drfloob) is the project lead, under advisement from Mark Roth
(@markdroth) and Craig Tiller (ctiller@). EventEngine implementers include
@dennycd, @nicolasnoble, @tamird, and @Vignesh2208. Python collaborators include
@gnossen and @lidizheng.

## TBD while the API remains experimental:

* Alterations to the Time type are expected.
* Finalized details on how custom EventEngine objects are provided.
* Finalized details on file descriptor support.

## Followup work

* Plans to deprecate the gRPC-Core-internal completion queue model.
