Core API to estimate the memory usage of channels and servers
----
* Author(s): apolcyn
* Approver: a11r
* Status: {In-Review}
* Implemented in: core
* Last updated: 1/18/2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/FPoXprcT0d4

## Abstract

Provide a way to estimate the memory usage of expensive objects in the C-core
like Channels and Servers.

## Background

Some wrapped languages have garbage collectors that act lazily and that
don't account for memory that was not allocated by the language's own
memory manager. For example, to the ruby garbage collector, a c-core Channel only
costs as much memory as the thin wrapper object itself, even though it might actually
end up allocating much more than that internally for different reasons.
This is problematic because this can prevent such a garbage collector from
cleaning things up in the face of rising memory usage. Having a core API that
gives an estimated memory cost would be useful to languages that have methods
of informing their garbage collectors about it.

### Related Proposals:
https://github.com/grpc/proposal/pull/54 is an explicit API that lets the user
releases resources explicitly.
The API in this proposal is different than #54 because this proposal helps to
inform the garbage collector specifically about memory usage. It could help
especially in e.g. the case in which there are many <i>live</i> channels
consuming large amounts of memory, and we want to inform the GC about it.

## Proposal

Provide an API on the c-core to estimate the memory usage of certain expensive
objects, especially channels and servers, and use it in wrapped languages that
have methods of informing their garbage collector's about external memory usage
(e.g., ruby's [rb_gc_adjust_memory_usage](https://bugs.ruby-lang.org/issues/12690) API).
