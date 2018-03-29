gRPC Channelz
====
* Author(s): Carl Mastrangelo
* Approver: a11r
* Status: Approved
* Implemented in: 
* Last updated: 2018-03-01
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/5IYOMVm0Ufs

Abstract
----
Adds channel level debug information that can be remotely queried.

Background
----

gRPC needs a way to expose internal details about connections.  This
information can be used to debug or diagnose a live program which may be
exhibiting undesirable network behavior.  In Stubby, this need is fulfilled by
the sub-pages of the /rpcz page, which exposes aggregated RPC stats as well as
listing all incoming and outgoing connections.  gRPC would like to include and
improve on this useful service.


# Channelz Service

The Channelz Service, hereafter channelz, reports all known channels, 
subchannels, servers, and connections that process currently has.  In a process,
there are multiple clients and servers.  To avoid confusion, a strict
terminology will be used:

1.  A "channel" represents an abstraction that can start and complete RPCs.  A
    channel may have channels and subchannels.  A channel is a directed 
    acyclic graph (DAG).  Only leaf channels may have
    sockets.  Interior channels may only have channels and subchannels.
2.  A "subchannel" represents an abstraction that is load balanced over by an 
    owning channel. A subchannel may have channels and subchannels.  A 
    subchannel is a directed acyclic graph (DAG).  Only leaf subchannels may 
    have sockets.  Interior subchannels may only have channels and subchannels.
3.  A "descendent channel" is either a channel or subchannel that is logically
    owned by a higher level channel.  Typically these are subchannels, but may
    be channels themselves. 
4.  A "server" represents the entry point for RPCs.  A server may have one or
    more listening sockets, and has a collection of "services".  Unlike 
    clients, servers are not hierarchical.  A server may only have sockets.
5.  An "endpoint" is either a channel or a server.  The local and remote
    endpoints may exist within the same process.
6.  A "socket" is roughly equivalent to a file descriptor.  A socket may be
    connected or unconnected (such as a listen socket).
7.  A "connection", sometimes called a transport, represents the link between
    two endpoints.  It is included here for completeness.
8.  A "client" is a role rather than an abstraction.  A client is the initiator
    of connections and RPCs, via its channels.  A client has channels, which it
    uses to send RPCs. This term is used less commonly.


# Channelz Data

The channelz service exposes internal implementation details in a public way.
Since different implementations have different internal details, the channelz
data is lightly abstracted.  There are four top level entities exposed:
Channels, Subchannels, Servers, and Sockets.

Each entity is uniquely identified by a positive integer, called the id.  This
id must be distinct and may not be reused over the lifetime of the process for
any entity.  This ensures that querying channelz will never result in referring
to the wrong entity.  While ids are not allocated sequentially, they are
ordered.  This enables pagination of the results when querying channelz.
Since protobuf treats the number 0 as the default value, it is unused as an id.

Along with the id, a human readable name can optionally be associated with each
the id.  The id and the name together form a "reference".  Thus, Channels,
Subchannels, Servers and Sockets are identified by their respective references.
These references are abbreviated ChannelRefs, SubchannelRefs, ServerRefs, and 
SocketRefs.  Note that only the id is necessary to query channelz.

The data representation of each ref:

![message ref structure][message ref structure]


## Channels and Subchannels

Channels and Subchannels, or descendent channels, are hierarchically organized
into a DAG structure.  The union of all channels and subchannels may not contain
a cycle.  A descendent channel may have any number of descendent channels. Each
descendent channel may also have any number of sockets.   However, a given 
descendent channel cannot have heterogeneous children.  That is, a channel or
subchannel may have descendent channels, or have sockets, but not both.

A subchannel represents a channel that is load-balanced over.  When a channel
has both subchannels and channels, the channels are not delegated to by the 
superchannel.  An example would be a channel that is used by the ancestor 
channel, but does not handle RPCs given by the application.

Each channel has a ChannelRef reference which includes the channel id and an
optional name.  The name is included for human consumption.  There are no
restrictions on the name but it should be limited to a reasonable length.

Each subchannel has a SubchannelRef reference which includes the subchannel id
and an optional name.  The name is included for human consumption.  There are 
no restrictions on the name but it should be limited to a reasonable length.

![channel hierarchy 1][channel hierarchy 1]

### Channel Data

Channels and Subchannels include data about themselves, notably their initial
parameters and their call stats.  Currently the data includes:

* Channel state ( See: [connectivity-semantics-and-api.md][connectivity-semantics] )
* Channel trace
* Channel target, if applicable.
* Number of calls started, succeeded, and failed.  Note this is slightly
  different than the number of streams.  See the Socket Data section.
* Time of the last call started.

In general, each piece of data included is specific to the channel itself and
NOT of its descendent channels.  This is to say that an ancestor channel
is not an aggregator for descendent channels.  The target of a descendent channel
may be different from that of its parent.  The target of a descendent subchannel 
may not be present.  The channel state may be different from the ancestor channel
state.   The number of calls started, succeeded, and failed should reflect if 
calls were specifically tied to the channel.

The number of calls started, succeeded, and failed on a channel gives insight
into the activity of the channel.  Subtracting the number of failed and
succeeded calls from the number started indicates how many are in flight.  The
total number of started calls indicates how active the channel has been over
time.  The ratio between succeeded calls and failed calls gives a rough
estimation of how healthy the channel has been.  The last call started timestamp
indicates how close the channel is towards entering the idle state.

#### Channel Trace

Channels and Subchannels also have a concept of a channel trace.  This includes
interesting events that have happened on the channel recently.  Example events
include entering different channel states, name resolution, load balancing
changes, unreachability, etc.  As mentioned earlier, this tracing info is
specific to the channel, and not an aggregation of descendent channel traces.

The channel traces may include ids pointing to channels and subchannels outside
of the current DAG.  Since channel traces use the same ids as the channels 
themselves, this allows a user to query for more information about a channel 
given its trace.

### Descendent channels

To keep the size of the Channel message down, only the ChannelRefs and 
SubchannelRefs will be included in the channel or subchannel.  These refs can
be used to walk the hierarchy.  Subchannels are not included in the top level
channels, and thus do not need to be paginated.  A user may query for all the
subchannels in parallel. As mentioned above, if there are any Sockets on the 
channel, there will be no descendent channels.   This invariant exists to 
simplify the tree structure and more closely match the known implementations of
gRPC.

The channel DAG represents a "has-a" rather than a "is-a" relationship.  In the
common case, there will be many subchannels.  These subchannels will be used to
balance RPCs for a given target.

Alternatively, a given channel's load balancer may itself make RPCs to gather
data about which subchannels to prefer.  Thus, the load balancing subchannels
are owned by a parent, but do not share its target.

![channel hierarchy 2][channel hierarchy 2]


### Sockets

Like channels and subchannels, only SocketRefs are included on a channel.  This 
is done for two main reasons.  The size of the socket message may be large, and
there may be many sockets for a given channel.  Secondly, it can be CPU 
intensive to fill in the socket information, even though the information might 
not be desired.  As mentioned in the section above, if there are any descendent 
channels, there will be no sockets.  This invariant exists to simplify the
DAG structure, and more closely match the known implementations of gRPC.

![channel hierarchy 3][channel hierarchy 3]


## Servers

Servers are conceptually simpler than channels, and thus do not have the same
hierarchical model.  There is no need for load balancing or name resolution.
Each "subchannel" is tied to a single socket, so the subchannel and socket are
flattened into just the socket.  Also, there are traditionally very few servers
running in a given process, and thus don't need a more complex model to express
their structure.

A server is an entry point for RPCs to be handled by a set of services.  The
group of services roughly defines a server, though services can be shared
between servers.  A server may be listening on any number of ports, though
commonly there is only one.  Each server has a ServerRef.  Like a ChannelRef, it
has a unique id and an optional name.   The id is unique, even among the ids of
channels, subchannels, and sockets.  This makes it impossible to accidentally 
refer to the wrong entity type when querying channelz.

Note that even with no Servers present, channelz data is still collected.  If in
the future a server is started, it can expose the channelz service which
includes prior channel information.

### Server Data

Server data includes data about the server, similar to Channel Data.  As of this
writing server data is a subset of the channel data:

* Number of calls started, succeeded, and failed.  Note this is slightly
  different than the number of streams.
* Time of the last call started

As before, the number of calls started, succeeded, and failed gives insight to
how healthy and active the server is.  The server does not start calls itself,
so the counts represent how many calls the remote endpoint has started.

### Server Channel Trace

Servers keep track of recent interesting events that have happened.  This
information is stored in the server channel trace.  The trace includes events
such as connecting clients, timeouts, protocol violations, etc.  It also
includes information about the shutdown of the server.  This is important as the
server may have many active calls when it is shutting down, which prevent it
from terminating.

### Sockets and Listen Sockets

Unlike channels, servers have two kinds of sockets.  A regular socket is one
that is connected to a remote endpoint.  This is the most common type of socket.
Servers also have a "listen" socket, which is used to accept and create new
sockets.

There may be no listen sockets, but active regular sockets.  This can occur
during shutdown, when the server no longer wishes to accept connections but
would like to gracefully complete the existing ones.  If a socket could exist in
both locations, it may.  Implementations should prefer to segregate the two
sockets.


## Sockets

Conceptually, sockets are the equivalent of a file descriptor.  They have a
local and remote address, as well as some concept of security detail.  Sockets
keep track of "streams" while Channels and Servers keep track of "calls."
Sockets also expose access to underlying details of the operating system, such
as socket options.  They are the lowest level in the channelz hierarchy.

A SocketRef is associated with each socket, and contains a unique id and an
optional name.  The id is unique among even the channels and servers.  The name
includes information about the socket in a human readable form.  This is
important as it is used as a "bread crumb" when deciding which socket to query
for, from the context of a channel or server.

### Socket Data

The data exposed by a socket is much lower level than the channel or server
data.  This information is meant to aid in debugging slow or stalled
connections.  By default it includes the following:

* Number of streams started, succeeded, or failed.
* Number of gRPC messages sent and received.
* Number of keepalive messages sent.
* The last time a outbound stream was started and the last time an inbound
  stream was started.
* The last time a outbound message was sent and the last time an inbound
  message was received.
* The amount of flow control window granted to the remote endpoint, and the
  amount of flow control window granted by remote endpoint.
* All known socket options set.

The stream counts differ slightly than that of the number of calls.  Calls may
be started before a socket (transport) is available, so the stream counter may
smaller than the corresponding call counters.  Conversely, if a stream fails and
the call is retriable, a new stream may be started.  This could make the stream
counts higher than the call counts.  Streams are considered successful if they
have the HTTP/2 EoS bit set, or failed if terminated otherwise (usually an
HTTP/2 RST_STREAM frame).  The number of successful streams may also be greater
than the number of successful calls.  For example, if the stream completes 
successfully, but the gRPC trailers are invalid.

The message counts represent how many gRPC messages are sent and received.
Calls may send and receive any number of messages, including zero.

The number of keepalives sent represents how often the local endpoint wants to
keep the socket alive.  Generally this is an HTTP/2 PING frame.  However, not
all PING frames are keep alives.  Specifically, pings that were sent as a reply
(ACK bit set) or were used for reasons unrelated to keep alives should not count
towards this.  For example, if pings were used to measure network bandwidth,
they would not be treated as keep alives.   The intent is to measure how often 
the local endpoint is doing extra work to keep the connection alive.


Like Channel data, the last time a stream was started by the remote or local
endpoint gives insight into the activity of the socket.  The number of locally
started streams is zero for server sockets, as servers do not start streams in
the gRPC protocol.

In the event of long lived streams, the last time a message was sent or received
is also useful to know.  The last timestamps are included for messages in both
directions.

Flow control is a very tricky subject and is often the source of hard to explain
slowdown.  The amount of window granted by each endpoint to each other is a
constantly evolving number, which each endpoint sending chunks of window.  For
the particular flow control windows in the socket data, they are the windows as
defined by the underlying transport.  For example, HTTP/2 transports report the
connection level flow control window.  Implementations that don't support flow
control may leave this blank, implying the limitation is either unknown or 
doesn't exist.  Note that only the connection level flow control window is 
reported, rather than the stream level.  TCP level flow and congestion control 
counters are exposed via socket options.

#### Socket Options

At this low level in the gRPC stack, implementation differences become more 
significant. Each OS or platform may have differing amounts of socket
information available.  To address these differences, each socket option name
and value is expressed as a string.

If there is a more structured form (such as TCP info), it too can be included
in the socket option message.  At least one of the string form or the structured
form should be present.  The additional data is exposed as a Proto3 Any field,
to accommodate types added later.  A few default messages are included for the
common options, such as that for TCP Info, and socket timeouts.

When querying for socket option data, as much detail should be included as
possible.  Since there are many types of socket options (at least 50),  the
number of options included in the socket data may be large.  If a querying a
socket option returns an error, it should not be fatal.  The channelz info is a
debugging tool, so it should try to ignore the failed socket option and continue
on.

This means that varying amounts of socket data may be returned by the channelz
service.  Implementations may avoid querying the options if it would be
impractical, though they are encouraged to do so.

### Addresses

An important part of debugging a connection is finding "who" the socket is
connected to.  To answer this question each socket exposes a remote and a local
address field.  The address implies what kind of socket it is (e.g. TCP, UDP,
UDS, etc.) as well as the the identity used to establish the connection (e.g.
IP Address, port number, filename, etc.).

Not every socket has a remote address;  listening sockets may only include a
local address.  Not every socket has a local address; Unix domain sockets may 
only include a remote address.

By default, only TCP and UDS addresses are defined.  Because there are many
kinds of address families that cannot be anticipated, the address includes an
extendable proto Any field for extension.

### Security

Cryptographically secure connections are expected for use with gRPC.  The
details of the crypto used on the connection are generally useful, and are thus
exposed on the Socket message.  As with Socket options and Addresses, there may
be many kinds of security that exist outside of the commonly used TLS, so the
structure is extensible.  By default, only a TLS message is defined.

# Querying Channelz

Channelz exposes four methods to access channelz data:

* GetTopChannels returns the root channels for channel trees
* GetServers returns the servers in the system
* GetChannel returns info about a specific channel
* GetSubchannel returns info about a specific subchannel
* GetSocket returns info about a socket

## Continuation Tokens

The first two methods, GetTopChannels and GetServers are based off the idea of
a continuation token.  A query for channels or servers includes a desired
"start" id, and requests all entities with an id greater than the given id.  The
response includes entities, in ascending id order up to a limit (by default
100).  When the client wants to request additional data, it sends another
request with the last id from the previous response.  This enables the user to
get the next available page of entities.  Thus, the id is the continuation token
that the user sends to traverse the entities.

This mechanism provides a reasonable expectation of visiting each entity.  As
entities are created and destroyed, their ids will remain stable.  Since ids are
never reused existing entities can't accidentally get skipped.  New entities may
be added before the current cursor, but all entities that existed at the time
scanning started can be reached.  Additionally, since the ids are stable and the
results sorted, other users can repeat the query and see the same page.

For example, the following sequence of events could occur: (C = client, S = server)
1.  C sends GetTopChannels(id=0)
2.  S sends RootChannels 1, 5, 9 but not 12, 13, 17, 18
3.  C sends GetTopChannels(id=9+1)
4.  S send RootChannels 12, 13, 17 but not 18
5.  C sends GetTopChannels(id=17+1)
6.  S sends RootChannels 18, end=true

In this exchange, the client C requests some pages starting with the invalid
id 0.  The server sends back a page with 3 results.  After the user looks at the
first three items, the last item's id plus one is requested (9 + 1 = 10).  While
there is no channel with id 10, the server S sends back the first 3 items
greater than 10.  The client repeats the query with the new latest number, but
gets back a response where the server indicated there were no more results.
While a human user could retry the query, perhaps to see if new channels have
been created, an automated user would know to stop here.

The channelz service may return any number of entities, though the results
should be capped to a reasonable number.  If the process is under heavy load,
it may decide to return fewer entities.  Since there is a flag in the response
indicating that the user is at the end, channelz may even return an empty list.
All ids are valid start points, including non positive ids.

## GetTopChannels

This method returns all the root channels in the process.  Users are expected
to make this query first when investigating channels.  As mentioned in the
previous section, channelz will respond with a list of zero or more top level
channels, as well as a boolean flag indicating if there may be more results.
Users who wish to get the next results should use the largest channel id from
the previous results in the subsequent query.

## GetServers

This method returns all servers in the process.  The usage is identical to GetTop
Channels, except that servers are returned.  Users are expected to make this
query first when investigating servers.  Since servers are not hierarchical,
there is no GetServer method.

## GetChannel

This method returns a single Channel request by its id.  If the channel is not
present, a `NOT_FOUND` status is returned.  Users are expected to make this
query when traversing a single channel tree.

## GetSocket

Like GetChannel, this returns a single Socket, or a `NOT_FOUND` if there is no
socket with the requested id.

# Alternatives Considered

There are a number of alternative possible ways to structure and expose channel
data.  Many were considered and discussed before reaching the current state.
Some of the more notable alternatives are listed here.

## Static or Dynamic Channel Trees

In the previous model (that of Stubby) channel structure was highly static.
The hierarchy for channels was either 2 or 3 nodes deep, depending on if the
channel used load balancing.  This model is conceptually simpler to think about,
as the tree depth is tightly bounded.  This model had not been changed for many
years, indicating that it was unlikely to change in the future.

While the previous model has served well, it is not sufficient for future use.
As gRPC is an open source project with an open specification, there will be many
implementations.  These implementations may need to express arbitrary channel
hierarchies in ways that cannot be anticipated.  Additionally, gRPC itself is
already not quite a perfect fit for this model, as channels to the loadbalancer
are owned by the root channel.  Since they are not technically subchannels of
the root channel, they do not fit in the Channel > Subchannel > Socket
structure.

Additionally, subchannels are practically identical to channels; the only
difference is there location in the tree.  Even though they would have had a
specific message type, it didn't make sense to have two substantially equivalent
messages.

## Servers and Channels or just Channels

In the existing implementations of gRPC, servers are implemented using channels.
There are not many behavioral differences between clients and servers for a
given connection.  They both speak the same protocol.  Many of the fields
overlap between the two types, and it could be conceptually simpler to unify
them into one type.  For fields that don't apply to one or the other, the field
could be left unset.  There is also a fair amount of flexibility in the current
model with regards to channel structuring, why place an arbitrary division
between channels and servers?

There are several reasons, experience has shown, to have a more separated model
when it comes to overlapping entities.

When it comes to selectively setting or checking fields, it may not be clear
from the documentation if the field should be accessible.  Some fields may be
sometimes set, but users may use them anyways.  For example a server doesn't
have a single "target" it is connected to, but with a unified structure it would
not be clear if the missing target was a buggy channel or a server.  Another
example is the number of retries on a given channel.  If there were zero, was it
because it was a server, or because it was a channel that simply didn't retry?

Having more concrete message types makes the proto definition more self
documenting.  While comments could explain when which fields are set, having
distinct types lessens this need.  Fundamentally, the specification tells what
should be produced rather than what is possible.   This makes it easier to do
the right thing, and harder to make mistakes.  While the comments about how to
use the overloaded field could also exist outside the proto (such as a gRFC),
protos are the de facto source of truth.

Lastly, code that renders channelz data must be prepared to handle arbitrary
tree structures.  This implies that interpreting this data is already a
burdensome task; unifying channels and servers does not make the task more
difficult.  While this is true, implementers too must decide which fields must
be set.  Since they cannot know how all readers might interpret the data, they
cannot easily decide which fields can be safely set or unset. (Is it safe to
stop setting an arbitrary field?)

## Page-Offset or Continuation Token Pagination

Pagination of results makes it possible to load just some of the large amount of
data.  A simple way to do this is to to include the page number and the size of
the page to jump to the specific page of results.  Users of the pages can think
of them as actual book pages (i.e. the "nth" page).  Implementers can easily
retrieve the results since the offset can be calculated in constant time.

The problems with this approach are numerous, and readily visible in the
previous system.  Mainly,  there is unstated but wrong assumption: the channel
set doesn't change.  Channels can be created and destroyed concurrently while
the results are being paged through.  This means that page 1 will have its
results constantly changing on active systems.  Trying to share the page with
someone else risks them seeing a completely different result set.  When
retrieving page 2, the channels on the previous page may have been deleted.
This causes the results to "slide" backwards, increasing the risk of missing
some results.

A second problem is that it implies random access.  To get a page of 100
results, starting at page 50, the channelz service must get channels 5000
to 5100.  If channels are not stored in an array, this is significantly more
difficult to do.   Storing the results in a hashtable or binary tree would
require much more bookkeeping.

A third problem is that it hugely limits the protocol in regards to filtering.
Suppose in the future a parameter is added to the GetTopChannels request to only
include active channels which were made from a particular network interface.
The channelz service would have to construct a potentially expensive result set,
and throw away all but the last results that the user requested.

A fourth problem is that there is no flexibility in page size.  The protocol
must either dictate a page size, or allow it to be configurable.  This puts an
unnecessary constraint on the server: it may be difficult to return a fixed
number of results per page.  For example, if a user requests a page size of a
thousand, should the server return a thousand results?  The server may not wish
to do so much work.

A fifth problem is that having a page size at all raises ambiguity.  For
example, if the page size is 100, and the user is on page 50, but the user wants
to change to using a smaller page size like 25, there isn't a good way to do so.
The offset into the result set is now conflated with how many results are
desired.

These problems are ameliorated by using a continuation token.  A continuation
token is traditionally an opaque identifier included in the response by a
server.  The user includes this token on subsequent requests to get the next
results.  Though, in the case of channelz, the identifier is not opaque.

The first problem is not solved by the token, but it is better.  If a request
is made at time t0, and subsequent requests are made at time tN, then the client
can be certain to visit every channel that existed at time t0 and still exists
at tN.  This allows older channels to be reachable even as new ones are created.
There is no possibility of missing a channel that did exist since the user
remembers exactly where the last results were.  A subsequent iteration through
the pages will be certain to find newly created channels.  Compare this to the
page-offset approach, which could conceivably never find a particular channel.

The second problem is also solved.  Since the ids are integers, the can be
hashed or binary searched.  This is not an unreasonable performance penalty,
and leaves more flexibility in implementations.

The third problem is now easier too.   If query parameters are added in the
future, the server could keep a snapshot around for such results rather than
recalculate the results each query.  The token is where to resume from.

The fourth and fifth problem are solved now too.  The number of results can be
variable, allowing the server to return fewer results if convenient.  The client
can optionally include a desired page size, which the server can honor if
acceptable.

## ServerChannels or Sockets

Should servers have Subchannels? The semantics of servers are different than
channels, but not by a lot.   As mentioned in the "Servers And Channels or just
Channels" alternative, servers can be considered a kind of channel (though, as
the section explains, it was decided to keep them separate).  One alternative
that can address this similarity are ServerChannels.  ServerChannels would act
like a subchannels, but represent call level information specific to a server
channel.  In the current hierarchy, there is only room for servers to record
stream level info as opposed to call level.

While not being able to store call level information is unfortunate, it isn't
so bad.   For example, each socket that a server has encompasses all information
that a channel does.  Each stream corresponds to a call.

## Alternative IDs

There are many alternative ways of specifying ids.  Each of the id schemes
listed below has some shortcoming that makes it unsuitable for implementors or
consumers.

### Keep ids unique, but allow for reuse

This scheme does work, but raises the risk for ids from one part of the system
to point to out of date info.  For example, the ChannelTrace info will contain
ids of other entities.   If the ids could be reused, the trace info could refer
to the wrong entity.

### Keep ids unique, but silo them based on entity type

This scheme allows channel 1 to coexist with server 1.  There are two problems
with this scheme.  First, ids serve an additional purpose.  When looking through
logs, it is convenient to search based on the id of the entity being logged.  If
ids could be in each entity silo, it would make debugging more difficult.  The
second problem is more subtle.  Humans may type in the id into a system.  By
making ids globally unique, it becomes impossible to put a server id in a
channel field.

### Use opaque ids

This would be something like using surrogate key, perhaps a UUID, to identify
the entity.  This allows flexibility to change the implementation without
affecting the interface.  The problem with this is that it makes the
implementor's life easier, but not the consumer's!  It is useful to be able to
cross reference ids from channelz to logs to even a live running system.

### Use hierarchical ids

Instead of looking up channels and servers by their individual ids, one idea is
to express the lineage in the id.  For example, if channel 2 is a child of
channel 1, its key could be "1-2".  This would allow fast lookup, and allow
validation that the node requested was actually the one returned.  The pain
point for this scheme is that it is more complicated to implement for not much
benefit.  There is a minor downside too: it means entities can't be reparented
if the implementation ever wanted to.   No known implementations do this, but it
isn't forbidden.  Specifying the full hierarchy in the key would give rise to
errors about the channel not being found.

[message ref structure]: A14_graphics/1.png "Message Reference Structure"
[channel hierarchy 1]: A14_graphics/2.png "Channel Hierarchy"
[channel hierarchy 2]: A14_graphics/3.png "Channel Hierarchy"
[channel hierarchy 3]: A14_graphics/4.svg "Channel Hierarchy"
[connectivity-semantics](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md)


## Proto

```proto
syntax = "proto3";

package grpc.channelz;

import "google/protobuf/any.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

// Channel is a logical grouping of channels, subchannels, and sockets.
message Channel {
  // The identifier for this channel.
  ChannelRef ref = 1;
  // Data specific to this channel.
  ChannelData data = 2;
  // At most one of 'channel_ref+subchannel_ref' and 'socket' is set.
  
  // There are no ordering guarantees on the order of channel refs.
  // There may not be cycles in the ref graph.
  // A channel ref may be present in more than one channel or subchannel.
  repeated ChannelRef channel_ref = 3;
  
  // At most one of 'channel_ref+subchannel_ref' and 'socket' is set.
  // There are no ordering guarantees on the order of subchannel refs.
  // There may not be cycles in the ref graph.
  // A sub channel ref may be present in more than one channel or subchannel.
  repeated SubchannelRef subchannel_ref = 4;
  
  // There are no ordering guarantees on the order of sockets.
  repeated SocketRef socket_ref = 5;
}

// Subchannel is a logical grouping of channels, subchannels, and sockets. 
// A subchannel is load balanced over by it's ancestor
message Subchannel {
  // The identifier for this channel.
  SubchannelRef ref = 1;
  // Data specific to this channel.
  ChannelData data = 2;
  // At most one of 'channel_ref+subchannel_ref' and 'socket' is set.
  
  // There are no ordering guarantees on the order of channel refs.
  // There may not be cycles in the ref graph.
  // A channel ref may be present in more than one channel or subchannel.
  repeated ChannelRef channel_ref = 3;
  
  // At most one of 'channel_ref+subchannel_ref' and 'socket' is set.
  // There are no ordering guarantees on the order of subchannel refs.
  // There may not be cycles in the ref graph.
  // A sub channel ref may be present in more than one channel or subchannel.
  repeated SubchannelRef subchannel_ref = 4;
  
  // There are no ordering guarantees on the order of sockets.
  repeated SocketRef socket_ref = 5;
}

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

message ChannelData {

  ChannelConnectivityState state = 1;

  // The target this channel originally tried to connect to.  May be absent
  string target = 2;
  
  ChannelTrace trace = 3;

  // The number of calls started on the channel
  int64 calls_started = 4;
  // The number of calls that have completed with an OK status
  int64 calls_succeeded = 5;
  // The number of calls that have a completed with a non-OK status
  int64 calls_failed = 6;

  // The last time a call was started on the channel.
  google.protobuf.Timestamp last_call_started_timestamp = 7;
}

message ChannelTrace {
  // see the proto in the gRFC for channel tracing:
  // A3-channel-tracing.md
}

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

message SocketRef {
  int64 socket_id = 3;
  // An optional name associated with the socket.
  string name = 4;
  // Intentionally don't use field numbers from other refs.
  reserved 1, 2, 5, 6, 7, 8;
}

message ServerRef {
  // A globally unique identifier for this server.   Must be a positive number.
  int64 server_id = 5;
  // An optional name associated with the server.
  string name = 6;
  // Intentionally don't use field numbers from other refs.
  reserved 1, 2, 3, 4, 7, 8;
}

message Server {
  ServerRef ref = 1;
  ServerData data = 2;

  // The sockets that the server is listening on.  There are no ordering
  // guarantees.
  repeated SocketRef listen_socket = 3;
}

message ServerData {
  ChannelTrace trace = 1;
  
  // The number of incoming calls started on the server
  int64 calls_started = 2;
  // The number of incoming calls that have completed with an OK status
  int64 calls_succeeded = 3;
  // The number of incoming calls that have a completed with a non-OK status
  int64 calls_failed = 4;

  // The last time a call was started on the server.
  google.protobuf.Timestamp last_call_started_timestamp = 5;
}

// Information about an actual connection.  Pronounced "sock-ay".
message Socket {
  SocketRef ref = 1;

  SocketData data = 2;
  // The locally bound address.
  Address local = 3;
  // The remote bound address.  May be absent.
  Address remote = 4;
  Security security = 5;

  // Optional, represents the name of the remote endpoint, if different than
  // the original target name.
  string remote_name = 6;
}

message SocketData {
  // The number of streams that have been started.
  int64 streams_started = 1;
  // The number of streams that have ended successfully with the EoS bit set for
  //  both end points
  int64 streams_succeeded = 2;
  // The number of incoming streams that have a completed with a non-OK status
  int64 streams_failed = 3;

  // The number of messages successfully sent on this socket.
  int64 messages_sent = 4;
  int64 messages_received = 5;

  // The number of keep alives sent.  This is typically implemented with HTTP/2
  // ping messages.
  int64 keep_alives_sent = 6;

  // The last time a stream was created by this endpoint.  Usually unset for
  // servers.
  google.protobuf.Timestamp last_local_stream_created_timestamp = 7;
  // The last time a stream was created by the remote endpoint.  Usually unset
  // for clients.
  google.protobuf.Timestamp last_remote_stream_created_timestamp = 8;

  // The last time a message was sent by this endpoint.
  google.protobuf.Timestamp last_message_sent_timestamp = 9;
  // The last time a message was received by this endpoint.
  google.protobuf.Timestamp last_message_received_timestamp = 10;

  // The amount of window, granted to the local endpoint by the remote endpoint.
  // This may be slightly out of date due to network latency.  This does NOT
  // include stream level or TCP level flow control info.
  google.protobuf.Int64Value local_flow_control_window = 11;

  // The amount of window, granted to the remote endpoint by the local endpoint.
  // This may be slightly out of date due to network latency.  This does NOT
  // include stream level or TCP level flow control info.
  google.protobuf.Int64Value  remote_flow_control_window = 12;

  repeated SocketOption option = 13;
}

message Address {
  message TcpIpAddress {
    // Either the IPv4 or IPv6 address in bytes.  Will either be 4 bytes or 16
    // bytes in length.
    bytes ip_address = 1;
    // 0-64k, or -1 if not appropriate.
    int32 port = 2;
  }
  // A Unix Domain Socket address.
  message UdsAddress {
    string filename = 1;
  }
  // An address type not included above.
  message OtherAddress {
    // The human readable version of the value.
    string name = 1;
    // The actual address message.
    google.protobuf.Any value = 2;
  }

  oneof address {
    TcpIpAddress tcpip_address = 1;
    UdsAddress uds_address = 2;
    OtherAddress other_address = 3;
  }
}

message Security {
  message Tls {
    oneof cipher_suite {
      // The cipher suite name in the RFC 4346 format:
      // https://tools.ietf.org/html/rfc4346#appendix-C
      string standard_name = 1;
      // Some other way to describe the cipher suite if
      // the RFC 4346 name is not available.
      string other_name = 2;
    }
    // the certificate used by this endpoint.
    bytes local_certificate = 3;
    // the certificate used by the remote endpoint.
    bytes remote_certificate = 4;
  }
  message OtherSecurity {
    // The human readable version of the value.
    string name = 1;
    // The actual security details message.
    google.protobuf.Any value = 2;
  }
  oneof model {
    Tls tls = 1;
    OtherSecurity other = 2;
  }
}

message SocketOption {
  string name = 1;
  // The human readable value of this socket option.  At least one of value or
  // additional will be set.
  string value = 2;
  // Additional data associated with the socket option.  At least one of value
  // or additional will be set.
  google.protobuf.Any additional = 3;
}

// For use with SocketOption's additional field.  This is primarily used for
// SO_RCVTIMEO and SO_SNDTIMEO
message SocketOptionTimeout {
  google.protobuf.Duration duration = 1;
}

message SocketOptionLinger {
  bool active = 1;
  google.protobuf.Duration duration = 2;
}

// Tcp info for SOL_TCP, TCP_INFO
message SocketOptionTcpInfo {
  uint32 tcpi_state = 1;

  uint32 tcpi_ca_state = 2;
  uint32 tcpi_retransmits = 3;
  uint32 tcpi_probes = 4;
  uint32 tcpi_backoff = 5;
  uint32 tcpi_options = 6;
  uint32 tcpi_snd_wscale = 7;
  uint32 tcpi_rcv_wscale = 8;

  uint32 tcpi_rto = 9;
  uint32 tcpi_ato = 10;
  uint32 tcpi_snd_mss = 11;
  uint32 tcpi_rcv_mss = 12;

  uint32 tcpi_unacked = 13;
  uint32 tcpi_sacked = 14;
  uint32 tcpi_lost = 15;
  uint32 tcpi_retrans = 16;
  uint32 tcpi_fackets = 17;

  uint32 tcpi_last_data_sent = 18;
  uint32 tcpi_last_ack_sent = 19;
  uint32 tcpi_last_data_recv = 20;
  uint32 tcpi_last_ack_recv = 21;

  uint32 tcpi_pmtu = 22;
  uint32 tcpi_rcv_ssthresh = 23;
  uint32 tcpi_rtt = 24;
  uint32 tcpi_rttvar = 25;
  uint32 tcpi_snd_ssthresh = 26;
  uint32 tcpi_snd_cwnd = 27;
  uint32 tcpi_advmss = 28;
  uint32 tcpi_reordering = 29;
}

service Channelz {
  // Gets all root channels (e.g. channels the application has directly
  // created). This does not include subchannels nor non-top level channels.
  rpc GetTopChannels(GetTopChannelsRequest) returns (GetTopChannelsResponse);
  // Gets all servers that exist in the process.
  rpc GetServers(GetServersRequest) returns (GetServersResponse);
  // Gets all server sockets that exist in the process.
  rpc GetServerSockets(GetServerSocketsRequest) returns (GetServerSocketsResponse);
  // Returns a single Channel, or else a NOT_FOUND code.
  rpc GetChannel(GetChannelRequest) returns (GetChannelResponse);
  // Returns a single Subchannel, or else a NOT_FOUND code.
  rpc GetSubchannel(GetSubchannelRequest) returns (GetSubchannelResponse);
  // Returns a single Socket or else a NOT_FOUND code.
  rpc GetSocket(GetSocketRequest) returns (GetSocketResponse);
}

message GetServersRequest {
  // start_server_id indicates that only servers at or above this id should be
  // included in the results.
  int64 start_server_id = 1;
}

message GetServersResponse {
  // list of servers that the connection detail service knows about.  Sorted in
  // ascending server_id order.
  repeated Server server = 1;
  // If set, indicates that the list of servers is the final list.  Requesting
  // more servers will only return more if they are created after this RPC
  // completes.
  bool end = 2;
}

message GetServerSocketsRequest {
  int64 server_id = 1;
  // start_socket_id indicates that only sockets at or above this id should be
  // included in the results.
  int64 start_socket_id = 2;
}

message GetServerSocketsResponse {
  // list of socket refs that the connection detail service knows about.  Sorted in
  // ascending socket_id order.
  repeated SocketRef socket_ref = 1;
  // If set, indicates that the list of sockets is the final list.  Requesting
  // more sockets will only return more if they are created after this RPC
  // completes.
  bool end = 2;
}

message GetTopChannelsRequest {
  // start_channel_id indicates that only channels at or above this id should be
  // included in the results.
  int64 start_channel_id = 1;
}

message GetTopChannelsResponse {
  // list of channels that the connection detail service knows about.  Sorted in
  // ascending channel_id order.
  repeated Channel channel = 1;
  // If set, indicates that the list of channels is the final list.  Requesting
  // more channels can only return more if they are created after this RPC
  // completes.
  bool end = 2;
}

message GetChannelRequest {
  int64 channel_id = 1;
}

message GetChannelResponse {
  Channel channel = 1;
}

message GetSubchannelRequest {
  int64 subchannel_id = 1;
}

message GetSubchannelResponse {
  Subchannel subchannel = 1;
}

message GetSocketRequest {
  int64 socket_id = 1;
}

message GetSocketResponse {
  Socket socket = 1;
}
```



