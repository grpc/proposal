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
    * [The Session Lifecycle](#the-session-lifecycle)
    * [Retries](#retries)
    * [Data Flow and Multiplexing (HTTP/2 on HTTP/2)](#data-flow-and-multiplexing)
    * [Alternative Wire Format](#alternative-wire-format)
    * [Why HTTP/2 on HTTP/2 is Better](#why-http2-on-http2-is-better)
    * [Server-Side Architecture: Dual Server Approach](#server-side-architecture)
  * [User API Support (Short-Term Workaround)](#user-api-support)
  * [C++ User-Facing APIs](#c-user-facing-apis)
    * [Client-Side API](#client-side-api)
    * [Server-Side API & Context Propagation](#server-side-api)
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

#### The Session Lifecycle

**Establishment and Barrier Mechanism:**
The session begins when the client initiates a Session RPC. To the client application, this returns a virtual stub immediately. To avoid extra round-trips, the client can start sending virtual RPCs immediately; these are queued internally on the server side until the server application explicitly signals that the session is ready. 

Alternatively, the client can wait for an explicit acknowledgement via the `OnSessionAcknowledged` reactor callback before sending vRPCs.

**Deadlines and Cancellations:**
* **Session Deadline**: Applying a deadline to the Session RPC propagates a timeout to the server. If this expires, the session handler immediately considers all live virtual requests cancelled with the same `DEADLINE_EXCEEDED` status.
* **Virtual RPC Deadline**: Deadlines on individual vRPCs work natively. When the deadline expires, the vRPC is canceled and the failure is sent as a payload message over the physical stream.

**Graceful Shutdown:**
The server can initiate a graceful shutdown of the session. When this occurs, the client virtual channel enters the `TRANSIENT_FAILURE` state, rejecting new vRPCs with `CANCELLED`, but allows any pending vRPCs to finish.

#### Retries

Retries for the session establishment request itself will work identically to regular RPCs. However, for virtual RPC failures, in general it is not advisable to use retries. Retries in general should happen to different backends. Since the session is tied to a single specific backend, retries should ideally be handled by the application across different sessions. That said, configuration options to retry on virtual channels will be provided for those who specifically need it.

#### Data Flow and Multiplexing (HTTP/2 on HTTP/2)

To natively inherit gRPC's rich feature set, we will build the virtual stack as a new HTTP/2 transport stack, over the standard HTTP/2 stack.

1. The client initiates a `SessionRequest`, establishing an outer HTTP/2 stream.
2. This outer HTTP/2 stream is wrapped in a new `SessionEndpoint` (an `EventEngine::Endpoint`).
3. An inner HTTP/2 transport is layered on top of this `SessionEndpoint`, establishing a new Virtual Channel.
4. Virtual RPCs sent on the Virtual Channel are encoded by the inner HTTP/2 transport into raw bytes and written to the `SessionEndpoint`. These raw bytes are transmitted as standard payload messages over the outer physical stream.

On the server side, receiving a `SessionRequest` similarly spins up an inner HTTP/2 transport that decodes the subsequent payload messages as independent inner streams. 

*(Note: When the server initiates a Graceful Shutdown as described in the Lifecycle section, gRPC will internally accomplish this by sending an HTTP/2 `GOAWAY` frame on the inner virtual HTTP/2 channel.)*

#### Alternative Wire Format

An alternative design considered was defining a custom frame format over the physical stream where the over the wire payload message will be structured as follows: Metadata Length (4 Bytes), followed by Serialized Metadata (an encapsulated client/server frame), and finally the Zero-Copy Raw Payload. 

In this alternative format, every vRPC message would explicitly include a `vrpc_id`, and the server session handler would parse the frame to dispatch it to the correct virtual request handler.

#### Why HTTP/2 on HTTP/2 is Better

The big benefit this approach provides is that we do not need to re-invent many gRPC features for virtual RPCs, which are already provided by the HTTP/2 stack.

If we utilized the custom alternative wire format, we would have to manually implement solutions for:
* **Flow Control:** In HTTP/2-on-HTTP/2, the HTTP/2 stream in the outer transport will proactively write data to the inner endpoint, freeing itself to allow the next messages (corresponding to other virtual RPCs) to go through. A combination of the inner transport's flow control window and max concurrent streams setting intrinsically prevents unbounded queue growth.
* **Fairness:** HTTP/2 implicitly handles chunking and interleaving frames, providing fairness and preventing a single large virtual RPC payload from blocking smaller vRPCs.
* **HPACK Compression:** Many requests have massive method names and small payloads. The inner HTTP/2 transport implicitly provides HPACK compression, caching repeated method names and drastically reducing transferred bytes.

#### Server-Side Architecture: Dual Server Approach

Often, gRPC servers are configured with mandatory interceptors or modules (e.g., authentication, authorization) that run on every incoming connection. Because virtual RPCs have already been authorized at the outer session level, re-running these modules on the inner virtual RPCs is unnecessary.

To allow virtual handlers to bypass these mandatory checks, the design uses a dual-server approach. We will create a new gRPC server without any listening port. Having no listeners means that it is not possible for this server to directly receive traffic from the external clients. Virtual service handlers are registered exclusively on this secondary server. When the primary gRPC server receives a Session Request, it bridges the inner HTTP/2 transport to this secondary server, ensuring vRPCs seamlessly bypass the primary server's mandatory connection-level interceptors.

### User API Support (Short-Term Workaround)

Because the `returns service` proto syntax will not be available until a future Protobuf edition, generated code for virtual stubs will not be immediately available. 

To bridge this gap, gRPC will provide a **Generic API** to allow clients in all languages to both initiate the session and invoke virtual RPCs without dependency on generated code.

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

**Context Propagation:**
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
