Channel Tracing
----
* Author(s): Noah Eisen (ncteisen@google.com)
* Approver: markdroth, ctiller
* Status: Draft
* Implemented in: N/A
* Last updated: 01/02/18
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/WFDj3KeHYTI

## Table of Contents

  * [Abstract](#abstract)
  * [Background](#background)
  * [Proposal](#proposal)
  * [Format of Exported Data](#format-of-exported-data)

## Abstract

The document proposes adding a dedicated, in-memory tracing object per channel and subchannel. The tracing object will record important steps in the life of a channel, like address resolution, subchannel creation, channel state changes, etc etc. The data from this tracing object will be made available through a new gRPC API as a JSON formatted string.

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs. Channel tracing and its parent project, [channelz](https://github.com/grpc/proposal/pull/40), will be exceedingly helpful for getting live channel data from a misbehaving program.

## Proposal

To implement channel tracing, we will introduce a dedicated, in-memory tracing object that is owned by each channel and subchannel. We will add an API that allows library users to retrieve all of the tracing object's data for the channel as a JSON formatted string. Implementations must expose a toggle for this behavior (for example, C++ will add a new channel argument, GRPC_ARG_CHANNEL_TRACING).

Since the tracing object exists in memory, care must be taken to prevent the tracing object from using too much memory. Implementations must allow setting a _max_trace_nodes_ variable to control how many nodes are allowed in any tracing object's node list.

At certain points a particular subchannel might stop being used. In order to efficiently discard old subchannels, we will use the following method: when the trace signaling the subchannel's death is about to be dropped from the parent channel, we will then discard and free the tracing object for that subchannel.

## Format of Exported Data

The data will be exported as JSON formatted string. The JSON must conform with the [protobuf-json mapping chart](https://developers.google.com/protocol-buffers/docs/proto3#json). The JSON schema will be as follows:

```
{
  "channelData": {
    "uuid": string,
    "numNodesLogged": number,
    "startTime": timestamp string,
    "nodes": [
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
