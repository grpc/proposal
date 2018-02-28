Channel Tracing
----
* Author(s): Noah Eisen (ncteisen@google.com)
* Approver: markdroth
* Status: Draft
* Implemented in: N/A
* Last updated: 02/09/18
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/WFDj3KeHYTI

## Table of Contents

  * [Abstract](#abstract)
  * [Background](#background)
  * [Proposal](#proposal)
  * [Format of Exported Data](#format-of-exported-data)

## Terminology

This design uses the same terminology discussed in the [channelz](A14-channelz.md) design. Channels and subchannels will be handled by the same channel tracing code, so for this design, a channel means either a channel or subchannel from the channelz terminology.

In addition, the following terminology will be used:

1.  a "Trace event" is an interesting thing that happens to a channel. Examples include things like creation, address resolution, subchannel creation, connectivity state changes.
2. "Trace data" is all of the data that is contained for one channel. This includes a list of _Trace events_, as well as metadata like the timestamp at which the channel was created.
3.  a "Tracing object" is an in-memory data structure responsible for holding the _Trace data_ for a single channel.

## Abstract

The document proposes adding a dedicated _Tracing object_ for every channel and subchannel. The _Tracing object_ will record important events in the life of a channel, like address resolution, subchannel creation, channel state changes, etc. The _Trace data_ from this _Tracing object_ will be made available through the [channelz service](A14-channelz.md).

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs. Channel tracing will be helpful for getting live channel data from a misbehaving program.

## Proposal

To support channel tracing, implementations will use a dedicated _Tracing object_ that is owned by each channel and subchannel. We will add an API that allows library users to retrieve all of the _Tracing object's_ data for the channel in a language idiomatic way. Implementations must expose a toggle for this behavior (for example, C++ will add a new channel argument, GRPC_ARG_CHANNEL_TRACING).

Since the tracing objects may consume large amounts of space, care must be taken to prevent the _Tracing object_ from using too many resources. gRPC implementations should allow limiting the number of _Trace events_ retained to a global maximum, as well as a per channel or subchannel maximum.

Implementations must keep the _Trace data_ for a subchannel until the _Trace event_ signaling that subchannel's destruction is overwritten or garbage collected in the last parent channel to reference it. After that point, implementations are free to use an appropriate garbage collection algorithm to reclaim the memory that the subchannel's _Tracing object_ was holding.


## Format of Exported Data

The data will be accessed via the channelz service, which sends a ChannelTrace proto as part of a larger message concerning a channel or subchannel. The following is a relevant excerpt, taken directly from the proto definition in channelz](A14-channelz.md).

```proto
message ChannelRef {
  // The globally unique id for this channel.  Must be a positive number.
  int64 channel_id = 1;
  // An optional name associated with the channel.
  string name = 2;
  // Intentionally don't use field numbers from other refs.
  reserved 3, 4, 5, 6;
}

message SubchannelRef {
  // The globally unique id for this subchannel.  Must be a positive number.
  int64 subchannel_id = 7;
  // An optional name associated with the subchannel.
  string name = 8;
  // Intentionally don't use field numbers from other refs.
  reserved 1, 2, 3, 4, 5, 6;
}

// A trace event is an interesting thing that happened to a channel or
// subchannel, such as creation, address resolution, subchannel creation, etc.
message ChannelTraceEvent {
  // High level description of the event.
  string description = 1;
  // Status/error associated with this event.
  // Optional, only present is Status != OK. 
  google.rpc.Status status = 2;
  // When this event occurred.
  google.protobuf.Timestamp event_timestamp = 3;
  // The connectivity state the channel was in when this event occurred.
  ChannelConnectivityState state = 4;
  // ref of referenced channel or subchannel.
  // Optional, only present if this event refers to a child object. For example,
  // this field would be filled if this trace event was for a subchannel being
  // created.
  oneof child_ref {
    ChannelRef channel_ref = 1;
    SubchannelRef subchannel_ref = 2;
  }
}

message ChannelTraceData {
  // Number of events ever logged in this tracing object. This can differ from
  // events.size() because events can be overwritten or garbage collected by
  // implementations.
  int64 num_events_logged = 3;
  // Time that this channel was created.
  google.protobuf.Timestamp channel_created_timestamp = 4;
  // List of events that have occurred on this channel.
  repeated ChannelTraceEvent events = 5;
}

message ChannelTrace {
  // Ref to the channel or subchannel this trace refers to.
  oneof ref {
    ChannelRef channel_ref = 1;
    SubchannelRef subchannel_ref = 2;
  }
  // All of the data for this channel.
  ChannelTraceData channel_data = 3;
  // Optional, only present if this channel has children
  repeated ChannelTrace child_data = 4;
}
```
