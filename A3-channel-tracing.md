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

The document proposes adding a dedicated _Tracing object_ for every channel and subchannel. The _Tracing object_ will record important events in the life of a channel, like address resolution, subchannel creation, channel state changes, etc. The _Trace data_ from this _Tracing object_ will be made available through a new gRPC API as a JSON formatted string.

Channel tracing is one aspect of the wider design, [channelz](A14-channelz.md).

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs. Channel tracing will be helpful for getting live channel data from a misbehaving program.

## Proposal

To implement channel tracing, implementations will use a dedicated, in-memory _Tracing object_ that is owned by each channel and subchannel. We will add an API that allows library users to retrieve all of the _Tracing object's_ data for the channel as a JSON formatted string. Implementations must expose a toggle for this behavior (for example, C++ will add a new channel argument, GRPC_ARG_CHANNEL_TRACING).

Since the tracing objects may consume large amounts of space, care must be taken to prevent the _Tracing object_ from using too many resources. gRPC implementations should allow limiting the number of _Trace events_ retained to a global maximum, as well as a per channel or subchannel maximum.

Implementations must keep the _Trace data_ for a subchannel until the _Trace event_ signaling that subchannel's destruction is overwritten or garbage collected in the last parent channel to reference it. After that point, implementations are free to use an appropriate garbage collection algorithm to reclaim the memory that the subchannel's _Tracing object_ was holding.


## Format of Exported Data

The data will be exported as JSON formatted string. The JSON must conform with the [protobuf-json mapping chart](https://developers.google.com/protocol-buffers/docs/proto3#json). The JSON schema will be as follows:

```

// This is the JSON schema for ref types (they occur multiple time in the JSON
// schema for ChannelTrace). Only one of channel_ref or subchannel_ref will be
// present in the ref. The (sub)channel_id field is a globally unique identifier
// to the channel or subchannel. The name field is a human readable string.
{
  "channel_ref": {
    "channel_id": int64,
    "name": string
  },
  "subchannel_ref": {
    "subchannel_id": int64,
    "name": string
  }
}

{
  "ref": { <ref type defined above> },
  "channel_data": {
    // Number of events ever logged in this tracing object. This can differ from
    // events.size() because events can be overwritten or garbage collected by
    // implementations.
    "num_events_logged": int64,
    // Time that this channel was created.
    "channel_created_timestamp": timestamp string,
    // List of events that have occurred on this channel.
    "events": [
      {
        // High level description of the event.
        "description": string,
        // Status/error associated with this event.
        // Optional, only present is Status != OK. 
        "status": enum string,
        "time": timestamp string,
        // can only be one of the states in connectivity_state.h
        "state": enum string,
        // uuid of referenced subchannel.
        // Optional, only present if this event refers to a child object.
        // and example of a referenced child would be a trace event for a
        // subchannel being created.
        "child_ref": { <ref type defined above> }
      },
    ]
  },
  // Optional, only present if this channel has children
  "child_data": [
    // List of child data, which is of the exact same format as this
    // entire JSON object.
  ]
}
```
