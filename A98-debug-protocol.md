A98: Debug protocol
----
* Author(s): ctiller
* Approver: markdroth
* Status: Draft
* Implemented in: <language, ...>
* Last updated: 2025/05/06
* Discussion at: https://groups.google.com/g/grpc-io/c/XrOzA4akIHo

## Abstract

Add a generalized debug interface for gRPC services.

## Background

In A14 we added channelz.
This protocol mixes some lightweight monitoring with some tracing and debuggability features.
It suffers from being relatively rigid in the topology it presents, and the set of node types available.


### Related Proposals: 
* A14 - channelz.

## Proposal

A new debug service proto will be added, `debug/debug.proto` in the `grpc.debug.v1` namespace.

### Entities

The fundamental building block of the protocol is a queryable entity.
An entity describes one live object inside of a gRPC implementation.
This might be a call, a channel, a transport, or a TCP socket. It might also describe a resource quota or a watched configuration file.

One entity is described as such:

```
message Entity {
    // The identifier for this entity.
    int64 id = 1;
    // The kind of this entity.
    string kind = 2;
    // Parents for this entity.
    repeated int64 parents = 3;
    // Instantaneous data for this entity.
    repeated google.protobuf.Any data = 4;
    // Historical trace information for the entity.
    repeated TraceEvent trace = 5;
}
```

Entities have state and configuration, and tend be be relatively long lived - such that querying them makes sense.

**id**: An entity is identified by an id.
These ids are allocated sequentially per the rationale in A14, and implementations should use the same id space for debug entities and channelz objects.

**kind**: An entity has a kind. This is a string descriptor of the general category of object that this entity describes.
We use strings here rather than enums so that implementations are free to extend the kind space with their own objects.
Common entity kinds will see some level of standardization across stacks, but we expect many kinds to be specific per gRPC implementation.
Initially implementations are expected to match kinds with channelz object types:
kind `"channel"` -> `channelz.v1.Channel`, `"subchannel"` -> `channelz.v1.Subchannel`, `"server"` -> `channelz.v1.Server`, `"socket"` -> `channelz.v1.Socket`.


**parents**: It's useful to be able to associate entities in parent/child hierarchies.
For example, a channel has many subchannel children.
Channelz listed specific kinds of children in its various node types - this tracking (and the need to produce it when sending an object) has caused contention issues in implementations in the past.
Should the list of children (optionally of a particular kind) for an entity be desired - and that's common - a separate paginated service call can be made.
Instead, this protocol lists only parents (as that set is far more stable).
Multiple parents are allowed - to handle at least the case of C++ subchannels being owned by multiple channels.

**data**: This is a list of protobuf Any objects describing the current state of this entity.
Implementations may define their own protobufs to be exposed here, and common sets will be standardized separately.

**trace**: Finally, an entity may supply a small summary of its history as a trace.
Each event is defined as:

```
message TraceEvent {
    // High level description of the event.
    string description = 1;
    // When this event occurred.
    google.protobuf.Timestamp timestamp = 2;
    // Any additional supporting data.
    repeated google.protobuf.Any data = 3;
    // Any referenced entities.
    repeated int64 referenced_entities = 4;
}
```

These are made available to all entities (in contrast to channelz that selected which nodes had traces, and which did not) - though an implementation need not make that facility available in its implementation of entities.

Also note that any notion of severity has been removed from the protocol (in contrast to channelz) - in practice this has not been a useful field.
When converting this protocol to channelz all trace events should be taken as CT_INFO.

Again, there is a facility for implementations to provide their own additional information in the **data** field.

### Queries

Queries are made available via the `Debug` service:

```
service Debug {
    // Gets all entities of a given kind, optionally with a given parent.
    rpc QueryEntities(QueryEntitiesRequest) returns (QueryEntitiesResponse);
    // Gets information for a specific entity.
    rpc GetEntity(GetEntityRequest) returns (GetEntityResponse);
    // Query a named trace from an entity.
    // These query live information from the system, and run for as long
    // as the query time is made for.
    rpc QueryTrace(QueryTraceRequest) returns (QueryTraceResponse);
}
```

**QueryEntities** allows the full database of entities to be queried for entries that match a set of criteria.
The allowed criteria may be extended over time with additional gRFCs.

```
message QueryEntitiesRequest {
    // The kind of entities to query.
    // If this is set to empty then all kinds will be queried.
    string kind = 1;
    // Filter the entities so that only children of this parent are returned.
    // If this is 0 then no parent filter is applied.
    int64 parent = 2;
}

message QueryEntitiesResponse {
    // List of entities that match the query.
    repeated Entity entities = 1;
}
```

**GetEntity** allows polling a single entity's state.

```
message GetEntityRequest {
    // The identifier of the entity to get.
    int64 id = 1;
}

message GetEntityResponse {
    // The Entity that corresponds to the requested id.  This field
    // should be set.
    Entity entity = 1;
}
```

Finally, **QueryTrace** allows rich queries of live trace data to be pulled from an instance.

```
message QueryTraceRequest {
    // The identifier of the entity to query.
    int64 id = 1;
    // The name of the trace to query.
    string name = 2;
    // The amount of time to query. Implementations may arbitrarily cap this.
    google.protobuf.Duration duration = 3;
    // Implementation defined query arguments.
    repeated google.protobuf.Any args = 4;
}

message QueryTraceResponse {
    // The events in the trace.
    repeated TraceEvent events = 1;
    // Number of events matched by the trace.
    // This may be higher than the number returned if memory limits were exceeded.
    int64 num_events_matched = 2;
}
```

The idea here is that these traces will be very detailed - beyond what can feasibly be stored in the historical traces.
So instead of needing to store this data in historical traces *in case* it is needed, we query and collect this data on demand for small windows of time.

### Well Known Data

A separate `well_known_data.proto` file will be maintained, with debug state that's common across implementations described within it.

The initial state will be:

```
// Channel connectivity state - attached to kind "channel" and "subchannel".
// These come from the specified states in this document:
// https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md
message ChannelConnectivityState {
    enum State {
        UNKNOWN = 0;
        IDLE = 1;
        CONNECTING = 2;
        READY = 3;
        TRANSIENT_FAILURE = 4;
        SHUTDOWN = 5;
    }
    State state = 1;
}

// Channel target information. Attached to kind "channel" and "subchannel".
message ChannelTarget {
    // The target this channel originally tried to connect to.  May be absent
    string target = 1;
}
```

## Rationale

The new interface is heavily inspired by channelz, but removes rigid node associations and predefined data sets.
Much of the infrastructure required for channelz can be shared with the debug interface, and indeed one can be implemented atop the other with good fidelity (in either direction).

Node ids are consistent between the protocols so that one can take a node id from the debug interface and query channelz data directly (and vice versa).
This will allow consumers of the protocols to gradually transition from one to the other.
It also allows re-using the book-keeping already in existence for channelz in implementations, without adding another kind of book-keeping to the mix.

Why not just extend channelz?

There are some fundamental data model issues in that protocol that I'd like to address going forward:

* chaotic-good (a transport currently implemented and in production in C++) has multiple TCP sockets per transport. This might be representable as a subchannel with multiple sockets on the client side (though we've already represented the HTTP2 transport as a socket there), but server side makes no allowance for properly describing the object hierarchy.
* Further, the set of node types ("kinds" here-in) and their relationships are pre-baked into the protocol, and as we evolve and improve gRPC new node types are needed and new relationships will be added or removed. We should not need to update channelz each time this happens.

Specific metrics have been not been carried forward from channelz as required parts of the protocol.
Users that need call or message counts from the system are encouraged to use the telemetry features of gRPC.
Implementations however are encouraged to publish whatever metrics they have available at query time to some `data` protobuf.

## Implementation

ctiller will implement this for C++. Other languages may pick this up as needed.
