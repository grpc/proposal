Channel Tracing
----
* Author(s): Noah Eisen (ncteisen@google.com)
* Approver: markdroth
* Status: Draft
* Implemented in: N/A
* Last updated: 01/02/18
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/WFDj3KeHYTI

## Table of Contents

  * [Abstract](#abstract)
  * [Background](#background)
  * [Proposal](#proposal)
  * [Format of Exported Data](#format-of-exported-data)

## Terminology

This design uses the same terminology discussed in the [channelz](A14-channelz.md) design.

In addition, the following terminology will be used:
1.  an "trace event" is an interesting thing that happens to a channel. Example include things like creation, address resolution, subchannel creation, connectivity state changes.
2. "trace data" is all of the data that is contained for one channel. This includes a list of trace events, as well as metadata like the timestamp at which the channel was created.
3.  a "tracing object" is an in-memory data structure responsible for holding the trace data for a single channel.

## Abstract

The document proposes adding a dedicated, in-memory tracing object for every channel and subchannel. The tracing object will record important events in the life of a channel, like address resolution, subchannel creation, channel state changes, etc etc. The trace data from this tracing object will be made available through a new gRPC API as a JSON formatted string.

Channel tracing is one aspect of the wider design, [channelz](A14-channelz.md).

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs. Channel tracing will be exceedingly helpful for getting live channel data from a misbehaving program.

## Proposal

To implement channel tracing, implementations will use a dedicated, in-memory tracing object that is owned by each channel and subchannel. We will add an API that allows library users to retrieve all of the tracing object's data for the channel as a JSON formatted string. Implementations must expose a toggle for this behavior (for example, C++ will add a new channel argument, GRPC_ARG_CHANNEL_TRACING).

Since the tracing object exists in memory, care must be taken to prevent the tracing object from using too much memory. Implementations must allow setting a _max_trace_events_ variable to control how many trace events are allowed in any tracing object's event list. Once the limit is reached, the oldest trace event will be overwritten by a new one (a circular buffer).

At certain points a particular subchannel might stop being used. However, the tracing object for that subchannel may still be needed if any trace events from other tracing object reference the subchannel (as in the case of a parent's subchannel being destroyed). In order to efficiently discard old subchannels, implementations must free the tracing object for a subchannel when the trace event signaling that subchannel's destruction is overwritten in the last parent channel to reference it.

## Format of Exported Data

The data will be exported as JSON formatted string. The JSON must conform with the [protobuf-json mapping chart](https://developers.google.com/protocol-buffers/docs/proto3#json). The JSON schema will be as follows:

```
{
  "channelData": {
    "uuid": string,
    "numEventsLogged": number,
    "startTime": timestamp string,
    "events": [
      {
        "data": string,
        "error": string,
        "time": timestamp string,
        // can only be one of the states in connectivity_state.h
        "state": enum string,
        // uuid of referenced subchannel.
        // Optional, only present if this event refers to a child object.
        // and example of a referenced child would be a trace event for a
        // subchannel being created.
        "child_uuid": string
      },
    ]
  },
  // Optional, only present if this channel has children
  "childData": [
    // List of child data, which is of the exact same format as the
  ]
}
```
