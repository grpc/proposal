Channel Tracing
----
* Author(s): Noah Eisen (ncteisen@google.com)
* Approver: markdroth
* Status: Draft
* Implemented in: N/A
* Last updated: 2018-03-01
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/WFDj3KeHYTI

## Terminology

This design uses the same terminology discussed in the [channelz](A14-channelz.md) design. Channels and subchannels will be handled by the same channel tracing code, so for this design, a channel means either a channel or subchannel from the channelz terminology.

In addition, the following terminology will be used:

1.  a "Trace event" is an interesting thing that happens to a channel. Examples include things like creation, address resolution, subchannel creation, connectivity state changes. Some _Trace events_ (like a new subchannel being created), will refer to the ChannelData of the relevant channel or subchannel.
2.  a "Channel trace" is a data structure responsible for holding all trace data for a single channel. This includes a list of _Trace events_, as well as metadata like the timestamp at which the channel was created.

## Abstract

The document proposes adding a dedicated _Channel trace_ for every channel and subchannel. The _Channel trace_ will record important events in the life of a channel, like address resolution, subchannel creation, channel state changes, etc. The data from this _Channel trace_ will be made available through the [channelz service](A14-channelz.md).

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs. Channel tracing will be helpful for getting live channel data from a misbehaving program.

## Proposal

The data from _Channel trace_ will be exposed via the [channelz service](A14-channelz.md).

Since the tracing objects may consume large amounts of space, care must be taken to prevent the _Channel trace_ from using too many resources. gRPC implementations must allow limiting the number of _Trace events_ retained to a per channel or subchannel maximum. Once the maximum is reached, adding a new _Trace event_ will cause oldest _Trace event_ to be removed. If the maximum number of trace events is set to zero, then channel tracing will be disabled.

The _Channel trace_ for a given channel or subchannel must be maintained as long as there are any _Trace events_ that refer to the channel or subchannel. After the last _Trace event_ that refers to the channel or subchannel is removed, the _Channel trace_ for that channel or subchannel may be cleaned up.

_Trace events_ should only be added for events that happen relatively infrequently (not at a per RPC basis). An example list of _Trace events_ from a healthy channel might look like:
```
... Channel created
... Address resolved: 8.8.8.8:443
... Address picked: 8.8.8.8:443
... Starting TCP connection
... TCP connection established
... Auth handshake complete
... Entering idle mode
```


## Format of Exported Data

The data will be accessed via the channelz service, which sends a ChannelTrace proto as part of a larger message concerning a channel or subchannel. The following is a relevant excerpt, taken directly from the proto definition in [channelz](A14-channelz.md).

```proto

// the definitions of these protos can be found in A14-channelz.md
message ChannelRef {}
message SubchannelRef {}

// A trace event is an interesting thing that happened to a channel or
// subchannel, such as creation, address resolution, subchannel creation, etc.
message ChannelTraceEvent {
  // High level description of the event.
  string description = 1;
  // The supported severity levels of trace events.
  enum Severity {
    CT_UNKNOWN = 0;
    CT_INFO = 1;
    CT_WARNING = 2;
    CT_ERROR = 3;
  }
  // the severity of the trace event
  Severity severity = 2;
  // When this event occurred.
  google.protobuf.Timestamp timestamp = 3;
  // ref of referenced channel or subchannel.
  // Optional, only present if this event refers to a child object. For example,
  // this field would be filled if this trace event was for a subchannel being
  // created.
  oneof child_ref {
    ChannelRef channel_ref = 4;
    SubchannelRef subchannel_ref = 5;
  }
}

message ChannelTrace {
  // Number of events ever logged in this tracing object. This can differ from
  // events.size() because events can be overwritten or garbage collected by
  // implementations.
  int64 num_events_logged = 1;
  // Time that this channel was created.
  google.protobuf.Timestamp creation_timestamp = 2;
  // List of events that have occurred on this channel.
  repeated ChannelTraceEvent events = 3;
}
```
