gRPC Debug Representation
----
* Author(s): Carl Mastrangelo
* Approver: a11r
* Status: Draft
* Implemented in:
* Last updated: 10/8/2018
* Discussion at: <google group thread>

## Abstract

Provide a common, terse, string representation of an RPC, useful for logging
or debugging.  This is different than the RPC Status, but may be included 
when logging.


## Background

It is occasionally useful to describe an ongoing or completed RPC for
consumption by humans.  There is no common representation currently that can
be used to express what an RPC _is_.  Other RPC systems provide a string
representation of the call, which contains the state of the RPC when queried.
This provides useful context when looking at RPC failures (or other problems),
since the `Status` is missing information about what operation failed.


## Proposal

It is proposed to make the RPC type include some minimal debug information 
that can be used to diagnose RPC failures.   This representation is 
primarily intended for non-automatic consumption and should be a short 
(&lt; 78 characters) string describing the call state.  The following values 
will be included (if not applicable, the values can be omitted.):

* Method name
* Time remaining before the deadline (ms)
* Original timeout for the call (ms)
* The socket id (per Channelz) of the call
* The trace id (per Census) of the call.
* (optional) Send/Receive state of the call.

Since this is intended for debugging, it is a design goal to make these values
easily searchable (such as using a Regex).  This means that the formatting of
the string needs to be precise and predictable. A subset of the protobuf text
format is suitable for this.  This means that the output should have exact 
spacing, limit the number of escape characters needed, fit on one line, have 
a predictable field ordering, and have a clear terminating character.  For 
example, an in progress call might look like:

```
method: "/my.qulaified.service/Method" timeleft_ms: 230 time_ms: 5000 channel: 53 trace: 9906
```

This would describe what the RPC is, how much time it started with, how long 
until the deadline is exceeded, how much time it had, what channel it is 
associated with, and what trace id identifies the call.

There is a lot of information intentionally not included.  The need to 
provide complete information needs to be balanced with what is useful to debug.
a full and explicit grammar can be specified once there is general agreement
for this proposal.

### Non-goals

It is not a goal to include this information in the status message.  Doing so 
would risk including internal details when propagting errors.  It is not a goal
to include all information about an RPC, even if it might have helped.  This is
an intentionally lossy representation of the RPC, with just enough info to 
retroactively debug.


## Rationale

### Why is this needed?

Debugging without context of the problem is inefficient.  Providing too little
information in gRPC makes it take a long time to diagnose, frequently 
involving:

* code modification 
* recompiliation with different flags
* live debugging of a failing process
* coordination between the service owner and gRPC team

While possible to do, this is slow, and increases the time before problems are
fixed.  Frequently, it is infeasible to modify the code to print out additional
debug info.  It may also be infeasible to recompile the code and redeploy as it
is time consuming.  If errors happen infrequently, it may be impossible to catch
the error live, making it impossible to diagnose with a debugger.  Lastly, 
debugging issues often requires heavy interaction with the gRPC team, which is 
not scalable, and not fast.  It would be ideal if service owners could debug 
their own services, without dependence on outside expertise.

### Why are current debugging tools not enough?

Too verbose logging drowns out useful information.  Logging all events on a 
channel is helpful for channel level events, but not RPC level events.
Modifying code is not always possible, and asking users to print out more
increases round trip time.  Statuses and status messages describe the failure,
but not the operation!

Consider the status: "8: method not implemented".  What method was not 
implemented?  Modifying the message to include the attempted method name 
still leaves out who the RPC was sent to.  What does 
"Code=4 "Deadline Exceeded" mean?  Was the deadline set too short
(perhaps 5ms instead of 5s)?   Does it mean that particular RPC timed out or
is the channel generally slow?   Without knowing what method was attempted,
it isn't easy to diagnose.


## Implementation

### Java 
ClientCall and ServerCall would get a toString() method that describes the info
above.  The StatusException and StatusRuntimeException would also be modified 
to include the call's string in their description, but not their Status.

### Go
TODO(dfawley): fill this in

### C++
TODO(yang-g): reassign or fill this in


