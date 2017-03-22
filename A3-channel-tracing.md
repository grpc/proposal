Channel Tracing
----
* Author(s): Noah Eisen (ncteisen@google.com)
* Approver: markdroth, ctiller
* Status: Draft
* Implemented in: N/A
* Last updated: 03/09/17
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/WFDj3KeHYTI

## Table of Contents

  * [Abstract](#abstract)
  * [Background](#background)
  * [Proposal](#proposal)
  * [Rationale](#rationale)
  * [Implementation](#implementation)
     * [Basic Design](#basic-design)
     * [Tracer Functions](#tracer-functions)
     * [API Changes](#api-changes)
     * [Format of Exported Data](#format-of-exported-data)
     * [Garbage Collection](#garbage-collection)

## Abstract

The document proposes adding a dedicated, in-memory tracing object per channel and subchannel. The tracing object will record important steps in the life of a channel, like address resolution, subchannel creation, channel state changes, etc etc. This trace will be made available with a new gRPC API function call. The long term goal is to incorporate the trace into grpcz or a custom channel tracing UI.

## Background

Channel connectivity issues are the root cause of a significant portion of user reported gRPC bugs (for example, [here](https://groups.google.com/a/google.com/forum/?utm_medium=email&utm_source=footer#!msg/grpc-users/gPFYbRAex1A/VM7h5FWAAgAJ) and [here](https://groups.google.com/a/google.com/forum/#!topic/grpc-users/0bsOqrxYvRc)). At the moment, gRPC has an extensive logging/tracing system in place. However, when juggling hundreds or thousands of connections, the logs can grow too huge to be quickly useful when debugging channel connectivity issues. Often times it is difficult to determine what events led to changes in connectivity.

## Proposal

To implement channel tracing, we will introduce a dedicated, in-memory tracing object that is owned by each channel and subchannel. We will add a new function to the grpc channel api that allows library users to retrieve all of the trace for the channel as a JSON or protobuf object. This returned trace will eventually be hooked into a UI, or exported through the ongoing grpcz project, where it could be used for debugging channel issues. The tracer will be enabled or disabled via a new channel argument, GRPC_ARG_CHANNEL_TRACING.

### Format of Exported Data

The trace data will be exported as JSON from the C-core. However, it will be JSON that is of the correct form to be transformed to protobuf and back again. The JSON schema will be as follows (note that all time strings will follow [RFC 3339](https://tools.ietf.org/html/rfc3339)):

```
{
  "channelData": {
    "numNodesLogged": number,
    "startTime": timestamp string,
    "nodes": [
      {
        "data": data,
        "error": string,
        "time": timestamp string,
        // can only be one of "GRPC_CHANNEL_INIT", etc etc 
        "state": enum string,
        // id of referenced subchannel
        "subchannelId": number
      },
    ]
  },
  "numSubchannelsSeen": number,
  "subchannelData": [
    {
      "subchannelId": number,
      "numNodesLogged": number,
      "startTime": timestamp string,
      "nodes": [
        {
          "data": data,
          "error": string,
          "time": timestamp string,
          "state": enum string,
        },
      ]
    },
  ]
}
```

This matches the protobuf form of the exported trace:

```protobuf
// one single node of trace data
message TraceNode {
  // static string describing what event the channel went through
  string data = 1;
  // error assosiated with the channel event
  string error = 2;
  // time the trace was logged
  google.protobuf.Timestamp time = 3;
  // unique id of subchannel associated with this trace
  // only used for trace present in the parent
  int64 subchannel_id = 4;

  enum ChannelState {
    GRPC_CHANNEL_UNKNOWN = 0;
    // channel has just been initialized
    GRPC_CHANNEL_INIT = 1;
    // channel is idle
    GRPC_CHANNEL_IDLE = 2;
    // channel is connecting
    GRPC_CHANNEL_CONNECTING = 3;
    // channel is ready for work
    GRPC_CHANNEL_READY = 4;
    // channel has seen a failure but expects to recover
    GRPC_CHANNEL_TRANSIENT_FAILURE = 5;
    // channel has seen a failure that it cannot recover from
    GRPC_CHANNEL_SHUTDOWN = 6;
  };
  // state the channel was in when the trace was logged
  ChannelState state = 5;
};

// trace data for one channel or subchannel
message TraceData {
  // unique id of subchannel associated with this trace data
  // set to zero if this is a channel's trace data
  int64 subchannel_id = 1;
   // tracks total trace nodes seen, since many may be overwritten
  int64 num_nodes_logged = 2;
  // time for creation for the channel or subchannel tracer
  google.protobuf.Timestamp start_time = 3;
  // all of the data for this tracer
  repeated TraceNode nodes = 4;
}

// trace for one channel
message ChannelTrace {
  // parent channel data
  TraceData channel_data = 1;
  // data for the subchannels of this channel
  repeated TraceData subchannel_data = 3;
}
```

## Rationale

This approach makes it easy to add trace, since each tracer is attached to its respective channel or subchannel. The tracer will also be passed down into the resolver and load balancer to record trace for interesting events that happen there. 

One downside to this approach is the potential for the tracer to bloat memory. For this reason, there will be a new channel argument that sets a limit for how many trace nodes each channel or subchannel will hold. This is described in more detail in the [Garbage Collection](#garbage-collection) section.

## Implementation

### Basic Design

The implementation will be done in the C-core first, with Java and Go to follow. The tracer object will hold a linked list of trace nodes. If a trace node pertains to a subchannel, then it will point to a tracer for that subchannel. tracers are refcounted objects.

```C
// One node of tracing data
struct grpc_trace_node {
  const char* data;
  grpc_error* error;
  gpr_timespec time_created;
  grpc_connectivity_state connectivity_state;
  grpc_trace_node* next;

  // The tracer object that owns this trace node. This is used to ref and
  // unref the tracing object as nodes are added or overwritten
  grpc_channel_tracer* subchannel;
};

struct grpc_trace_node_list {
  size_t size;
  size_t max_size;
  grpc_trace_node* head_trace;
  grpc_trace_node* tail_trace;
};

/* the channel tracing object */
struct grpc_channel_tracer {
  gpr_refcount refs;
  gpr_mu tracer_mu; 
  int64_t num_nodes_logged;
  grpc_trace_node_list node_list;
  gpr_timespec time_created;
};
```

### Tracer Functions

The tracer files with expose several calls which with the C-core code can interact with tracers:

```C
/* Initializes the tracing object with gpr_malloc. The caller has
   ownership over the returned tracing object */
grpc_channel_tracer* grpc_channel_tracer_create(size_t max_nodes);

/* Adds a new trace node to the tracing object */
void grpc_channel_tracer_add_trace(grpc_channel_tracer* tracer, 
    char* trace, struct grpc_error* error,
    grpc_connectivity_state connectivity_state);

/* Frees all of the resources held by the tracer object */
void grpc_channel_tracer_unref(grpc_channel_tracer* tracer);
```
### API Changes

And finally, a new method will be added to the channel surface:

```C
/* returns the tracing data in the form of a grpc json object */
grpc_json* grpc_channel_get_trace(grpc_channel* channel);
```

### Garbage Collection

In order to ensure that the tracer(s) do not use too much memory, both trace nodes and subchannel tracers will be discarded when the number of trace nodes has gotten too high. All tracers will only hold up to a configurable maximum number of trace nodes, the value of which can be set by setting a channel argument.

At certain points a particular subchannel might stop being used. In order to efficiently discard old subchannels, we will use the following method: when the trace signaling the subchannel's death is about to be dropped from the parent channel, we will then discard and free the tracer for that subchannel.
