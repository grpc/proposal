gRPC Virtual RPCs on Bidi Streams
---------------------------------

* Author(s): Siddharth Nohria
* Approver: a11r
* Status: Draft
* Implemented in: C++ (Prototype)
* Last updated: 2026-03-20

Table of Contents
-----------------

* [Abstract](#abstract)
* [Background](#background)
* [Terminology](#terminology)
* [Proposal](#proposal)
  * [Overview](#overview)
  * [Detailed Design](#detailed-design)
    * [Protobuf API](#protobuf-api)
    * [Generic API Support](#generic-api-support)
    * [The Session Lifecycle](#the-session-lifecycle)
      * [Establishment and Barrier Mechanism](#establishment-and-barrier-mechanism)
      * [Deadlines and Cancellations](#deadlines-and-cancellations)
      * [Graceful Shutdown](#graceful-shutdown)
    * [Retries](#retries)
    * [Data Flow and Multiplexing (HTTP/2 on HTTP/2)](#data-flow-and-multiplexing)
    * [Alternative Wire Format](#alternative-wire-format)
    * [Why HTTP/2 on HTTP/2 is Better](#why-http2-on-http2-is-better)
    * [Server-Side Architecture: Dual Server Approach](#server-side-architecture)
  * [C++ User-Facing APIs](#c-user-facing-apis)
    * [Client-Side API](#client-side-api)
    * [Server-Side API & Context Propagation](#server-side-api)
      * [Context Propagation](#context-propagation)
  * [Java User-Facing APIs](#java-user-facing-apis)
  * [Go User-Facing APIs](#go-user-facing-apis)

Abstract
--------

gRPC will support multiplexing virtual RPCs over a single common bi-directional stream. By establishing a logical "Session" over a physical stream, clients and servers can eliminate significant one-time setup costs associated with each RPC. This design improves end-to-end latency and CPU performance by caching and re-using heavy per-client application states across all virtual RPCs sent over the session.

Background
----------

Currently, each RPC sent over a gRPC channel incurs per-RPC overheads. For high-throughput services, operations like authentication, authorization policy validation, and other repetitive checks consume latency and CPU usage on a per-RPC basis. When the same clients are sending the RPC methods repeatedly, these are wasted cycles. Additionally, this allows applications to re-use heavy per-client application states, which are likely to be consistent across all RPCs from the same client.

While applications can implement custom multiplexing over bidirectional streams, doing so forces owners to reinvent core gRPC features such as telemetry and flow control. Moving this functionality into gRPC as a central framework avoids duplicated effort and provides a structured mechanism for high-performance, stateful RPC multiplexing.

Terminology
-----------

* **Physical Stream**: The underlying BiDi stream over which all messages are exchanged.
* **Virtual RPC (vRPC) / Stream**: The user-facing single RPC / stream, multiplexed over the physical stream.
* **Client / Server Session**: The internal gRPC state handling the virtual RPCs.
* **Session RPC**: The initial user-facing RPC used to establish and manage the session.
* **Virtual Stub**: Wraps a single physical stream and is used to start virtual RPCs.

Proposal
--------

### Overview

We propose introducing a "Session Request" RPC that establishes a long-lived physical bi-directional stream between the client and server. Rather than returning a standard response, this RPC yields a "Virtual Stub" on the client side. 

This virtual stub behaves structurally like a standard gRPC channel, allowing the application to invoke standard RPCs (Unary, Client/Server Streaming, Bi-directional Streaming). However, instead of creating new underlying connections or physical streams, these "Virtual RPCs" (vRPCs) are multiplexed over the single physical stream established by the Session Request. This amortizes connection setup and policy-checking overhead across all vRPCs in the session, improving throughput and latency.

### Detailed Design

#### Protobuf API

We introduce a new proto syntax: `returns service`. The client will send an initial Session Request to establish the common context to be used for every virtual RPC:

```proto
service FooService {
  rpc SessionRequest(ApplicationRequest) returns (service FooVrpcService);
}

service FooVrpcService {
  rpc VrpcUnaryMethod(VrpcRequest) returns (VrpcResponse);
  rpc VrpcClientStreamingMethod(stream VrpcRequest) returns (VrpcResponse);
  rpc VrpcServerStreamingMethod(VrpcRequest) returns (stream VrpcResponse);
  rpc VrpcBidiStreamingMethod(stream VrpcRequest) returns (stream VrpcResponse);
}
```

*Workaround*: Since the `returns service` syntax will not be available until a future Protobuf edition, the Session RPC can temporarily be modeled as a standard unary RPC that returns an `Empty` response, relying on manual channel configuration to bind the virtual service.

#### Generic API Support

To support proxy use cases where clients might not have compiled proto definitions, gRPC will provide a Generic API to allow clients in all languages to both initiate the session and invoke virtual RPCs without dependency on generated code.

#### The Session Lifecycle

##### Establishment and Barrier Mechanism

The session begins when the client initiates a Session RPC. To the client application, this returns a virtual stub immediately. 

To avoid extra round-trips, the client can start sending virtual RPCs immediately. However, the server application will provide an explicit signal (a "barrier") telling the session handler that all common application context has been successfully set up. Internally, this signal triggers sending the `server_initial_metadata` from the session request handler. 

Because the client does not need to wait for this signal, all virtual RPCs sent prior to the barrier are queued internally on the server side until this handshake is complete. Doing this queuing on the server side avoids adding an extra round trip before sending virtual requests, reducing latency for short-lived sessions. Alternatively, the client can explicitly wait for `server_initial_metadata` (via the `OnSessionAcknowledged` callback) before sending vRPCs.

##### Deadlines and Cancellations

We must support deadlines and cancellations for both the Session Request and each individual virtual request.

* **Session Deadline**: The session deadline is converted to a `grpc-timeout` header and sent to the server. If this expires, the gRPC client cancels the Session Request with a `DEADLINE_EXCEEDED` status immediately. The session handler then considers all live virtual requests cancelled with the same `DEADLINE_EXCEEDED` status and propagates this cancellation to the server. On the server side, the Session Handler sends a cancellation to each virtual handler before closing the physical stream.

* **Virtual RPC Deadline**: Deadlines on individual vRPCs work natively via the `ClientContext`. When the deadline expires, the event is caught by the session handler and sent as a payload message (encapsulating the timeout/cancellation) over the physical stream.

##### Graceful Shutdown

The server can initiate a graceful shutdown of the session, telling the client to finish all scheduled virtual RPCs but not start new ones. 

* Internally, gRPC accomplishes this by sending an HTTP/2 `GOAWAY` frame on the *virtual* channel.
* Upon receiving this, the client virtual channel enters the `GRPC_CHANNEL_TRANSIENT_FAILURE` state. New virtual RPCs are immediately rejected with `CANCELLED`.
* Once all pending virtual RPCs complete, the client sends a half close to the server and waits for trailing metadata before closing the session.

#### Retries

Retries for the session establishment request itself will work exactly the same as regular RPCs, using the same retry configuration. Because `server_initial_metadata` is not sent until successful establishment, all queued virtual RPCs can also be safely retried on the new physical stream. 

For virtual RPC failures, retries on the same session are generally not advisable because retries should typically happen to different backends. Since the session is tied to a single specific backend, retries should typically be handled by the application layer across different sessions. That said, configuration options to retry on virtual channels will be provided for clients who explicitly require it.

#### Data Flow and Multiplexing (HTTP/2 on HTTP/2)

To natively inherit gRPC's rich feature set, we will build the virtual stack as a new HTTP/2 transport stack, running *over* the standard HTTP/2 stack.

1. The client initiates a `SessionRequest`, establishing an HTTP/2 stream over the primary HTTP/2 transport. The initial metadata and first payload message are sent over this stream.
2. We wrap this established stream in a new gRPC Endpoint (e.g., an `EventEngine::Endpoint`).
3. We then create a new inner HTTP/2 transport layered on top of this Endpoint, and return the resulting Virtual Channel to the client application.
4. Virtual RPCs sent on the Virtual Channel are first sent to the inner HTTP/2 transport. The transport encodes these into raw bytes and writes them to the inner Endpoint. These raw bytes are then transmitted as standard payload messages over the outer stream.

On the server side, receiving a `SessionRequest` similarly spins up an inner HTTP/2 transport that decodes the subsequent payload messages as independent inner streams.

#### Alternative Wire Format

An alternative design considered was defining a custom frame format over the physical stream where the over the wire payload message would be structured as follows: Metadata Length (4 Bytes), followed by Serialized Metadata (an encapsulated `ClientVrpcFrame` or `ServerVrpcFrame`), and finally the Zero-Copy Raw Payload. 

In this alternative format, every vRPC message would explicitly include a `vrpc_id`, and the server session handler would parse the frame to dispatch it to the correct virtual request handler.

#### Why HTTP/2 on HTTP/2 is Better

The big benefit of the HTTP/2 on HTTP/2 approach is that we do not need to re-invent many gRPC features for virtual RPCs, which are already provided by the HTTP/2 stack.

If we utilized the custom alternative wire format, we would have to manually implement solutions for:

* **Flow Control:** gRPC’s standard flow control uses two windows: a connection-level window and a stream-level window, decided based on the Bandwidth Delay Product (BDP). In the HTTP/2-on-HTTP/2 model, the outer HTTP/2 stream proactively writes data to the inner endpoint, freeing itself to allow the next messages (corresponding to other virtual RPCs) to go through. A combination of the inner transport's flow control window and its `max_concurrent_streams` setting intrinsically prevents unbounded queue growth and prevents slow vRPC handlers from stalling the entire physical stream.

* **Fairness:** HTTP/2 implicitly handles chunking and interleaving frames, providing fairness and preventing a single large virtual RPC payload from blocking smaller vRPCs.

* **HPACK Compression:** Many requests have massive method names and small payloads. The inner HTTP/2 transport implicitly provides HPACK compression, caching repeated method names and drastically reducing transferred bytes.

#### Server-Side Architecture: Dual Server Approach

Often, gRPC servers are configured with mandatory interceptors or modules (e.g., authentication, authorization) that run on every incoming connection. Because virtual RPCs have already been authorized at the outer session level, re-running these modules on the inner virtual RPCs is unnecessary.

To allow virtual handlers to bypass these mandatory checks, the design uses a dual-server approach. We will create a new gRPC server without any listening port. Having no listeners means that it is not possible for this server to directly receive traffic from the external clients. Virtual service handlers are registered exclusively on this secondary server. When the primary gRPC server receives a Session Request, it bridges the inner HTTP/2 transport to this secondary server, ensuring vRPCs seamlessly bypass the primary server's mandatory connection-level interceptors.

### C++ User-Facing APIs

The C++ prototype uses Reactor-based APIs to manage the session lifecycle and establish virtual channels using the generic approach.

#### Client-Side API

Clients establish a session using `ClientSessionReactor` and `GenericStubSession`.

```cpp
class MySessionReactor : public grpc::ClientSessionReactor {
 public:
  void OnSessionReady(grpc::internal::Call call) override {
    // Create the virtual channel using the established call
    virtual_channel_ = grpc::experimental::CreateVirtualChannel(std::move(call));
    
    // Application can register state watchers here for Graceful Shutdowns
    // virtual_channel_->NotifyOnStateChange(...);
  }
  void OnSessionAcknowledged(bool ok) override { /* Session is fully ready */ }
  void OnDone(const grpc::Status& s) override { /* Session ended */ }
  
 private:
  std::shared_ptr<grpc::Channel> virtual_channel_;
};

// Setting up the session
grpc::experimental::GenericStubSession<SessionRequest, Empty> session_stub(channel);
auto* reactor = new MySessionReactor();
session_stub.PrepareSessionCall(&context, "/service/SessionRequest", {}, &request, reactor);
reactor->StartCall();
```

#### Server-Side API & Context Propagation

On the server, `ServerSessionReactor` is used to represent the lifecycle of the session. 

```cpp
class MyServerSessionReactor : public grpc::ServerSessionReactor {
 public:
  MyServerSessionReactor() {
    // Notify gRPC that we are ready to accept virtual RPCs
    StartVirtualRPCs();
  }
  void TriggerGracefulShutdown() {
    InitiateGracefulShutdown([this](absl::Status status) {
       Finish(grpc::Status::OK);
    });
  }
  void OnDone() override { delete this; }
};
```

##### Context Propagation

To share state (e.g., Auth/Security results), the Server application sets context on the parent session's arena. The virtual handler can easily access the application context via the session arena pointer stored in the virtual call arena:

```cpp
// In Session Handler: Set the context
context->SetContext(std::make_shared<ApplicationContext>(...));

// In Virtual RPC Handler: Retrieve the shared context
auto app_context = context->GetSessionContext<ApplicationContext>();
```

### Java User-Facing APIs

TBD

### Go User-Facing APIs

TBD
