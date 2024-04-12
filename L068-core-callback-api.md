Core callback-based completion queue
----
* Author(s): vjpai
* Approver: markdroth
* Status: Approved
* Implemented in: https://github.com/grpc/grpc/projects/12
* Last updated: June 9, 2020
* Discussion at https://groups.google.com/g/grpc-io/c/3peKeX7fXvY

## Abstract

Provide a completion-queue implementation for gRPC core in which the
completion of an operation triggers the invocation of a registered
piece of user code rather than posting a tag on an actual queue. The
new completion queue will have no explicit retrieval function at the
surface API level.

## Background

gRPC Core uses a completion queue as a notification mechanism of
completion of RPC stream operations.  `grpc_call_start_batch` (the
surface API for initiating RPC operation batches) and certain other
functions associate an RPC operation
with a _tag_ (currently-unique ID) when it starts. The tag is of type
`void*` but is not interpreted by gRPC core. The core API user
(a gRPC wrapped language implementation) will later see that tag when
accessing the completion queue via a specified accessor function.
There have long been 2 completion
queue types:

* `GRPC_CQ_NEXT` : arbitrary number of operations may be queued to it,
accessed through `grpc_completion_queue_next` which returns the "next"
item in the queue (but not necessarily FIFO with respect to wall-clock
time, initiation order, etc.)
* `GRPC_CQ_PLUCK` : limited number of operations may be queued to it,
accessed through `grpc_completion_queue_pluck` which allows waiting
for a specific tag to arrive at the queue and provides that specific
tag

The completion queue accessor functions may block until a specified
deadline, and during that time the user's thread is available for the
library to use in performing RPC operations or polling the operating
system for the completion of operations.

### Related proposals

* An asynchronous event processing loop implementation is critical to
the operation of the callback completion queue since this
completion queue type will not provide any accessor function to drive
the progress of the library. Python achieves this
through its asyncio interface, but more generally, the
[EventManager](https://www.github.com/grpc/proposal/pull/182) is
gRPC's multi-lingual answer to this problem.
* The callback completion queue is the implementation mechanism for the [C++ callback API](https://www.github.com/grpc/proposal/pull/180)

## Proposal

gRPC core will now support a new completion queue type called
`GRPC_CQ_CALLBACK`. Although this is called a completion queue because
it supports the semantic interface of `grpc_completion_queue`,
operations will never be queued in this structure. Instead, a
completion will cause a user-specified function to be called.

The tags injected into this completion queue
are still of type `void*` so as not to affect the API of RPC operation
initiation functions, but these will be interpreted by the
library. These tags *must* point to a C89 struct of type
`grpc_completion_queue_functor` which has two user-provided members:

* `void (*functor_run)(struct grpc_completion_queue_functor*, int)` : a pointer to a
function that will be invoked when the operation is completed. The
function must not have a return value and will take two arguments:
the functor object passed in as a tag and a boolean (represented as
a C89 int) specifying whether or not the operation completed
normally (corresponding to the `ok` result of
`grpc_completion_queue_next` on the `GRPC_CQ_NEXT` queue).
* `int inlineable`: specifies whether this functor can be run inline
in the same thread as the one that detected it rather than sending
it to an executor thread. This must be
used very carefully and only for known-safe callbacks that
do not use locks. Incorrect use can lead to deadlock, so this flag must always
be set to `0` (false) unless the library is certain that the callback always
meets the requirements. In practice, it should only be used for code that
belongs to a library that wraps core (e.g., the `DefaultReactor` in the C++ 
callback API implementation) and not for end-user application callbacks.

### Usage example

As before, `grpc_call_start_batch` is the most common entry point for
posting a new operation in gRPC, and one of its arguments is the completion
queue tag associated with the operation. For this type of completion-queue,
the tag must actually point to a `grpc_completion_queue_functor`. Since this
structure has minimal useful content, it should generally be used either via
composition or inheritance. (Existing gRPC libraries use both formats, with
Python using composition and C++ using inheritance.) The following is an
example of hypothetical usage in C using composition.

```
struct MyFunctor {
  grpc_completion_queue_functor functor;  // first field to match pointer
  int my_other_field1;
  char* my_other_field2;
  ...
};

void RunFunction(grpc_completion_queue_functor* functor_arg, int ok) {
  MyFunctor* functor = (MyFunctor*)functor_arg;
  printf("Operation %d of type %s complete\n", functor->my_other_field1,
          functor->my_other_field2);
  ...
  free(functor);
}

void InitiateOperation() {
  MyFunctor* functor = malloc(sizeof(MyFunctor));
  
  functor->functor.functor_run = &RunFunction;
  functor->functor.inlineable = 0;
  functor->my_other_field1 = 18;
  functor->my_other_field2 = "hello world";

  grpc_call_start_batch(..... , functor);
}
```

## Rationale

A more thorough longer-term solution would have been to use a
notification mechanism specific to the C++ callback API rather than trying
to wedge this implementation into the existing completion queue
definition. That said, such an approach would likely further delay the
deployment and development of the C++ callback API. The approach
described in this proposal naturally supports the implementation of the C++ callback
API with minimal extensions to the core surface and no core surface
API breakages.

In the longer-term, we expect to take advantage of the
[EventManager](https://github.com/grpc/proposal/pull/182) integration
more explicitly and use direct callbacks via the EventManager rather
than CQ-based notifications. Full adoption of the EventManager and the
elimination of the existing thread-borrowing polling model will enable
the removal of `grpc_completion_queue` from the core surface
API. However, that is a longer-term project that needs to be planned separately.

## Implementation

The implementation consists of two components: a new completion queue
implementation and a thread-local optional execution component called
the `ApplicationCallbackExecCtx`. The internal function
`grpc_cq_end_op` is implemented as `cq_end_op_for_callback` that
interprets the user tag as a `grpc_completion_queue_functor` and sends
it to an executor thread (normally) or runs it in the same thread by enqueuing it
on this thread's `ApplicationCallbackExecCtx` if possible.

The `ApplicationCallbackExecCtx` is a thread-local queue to which
functors can be queued and which runs those functors one at a time
when it goes out of scope. Just like `ExecCtx`, this is normally used
at all entry points to gRPC core. However, it is technically an
optimization and the code will work fine if it is not actually
instantiated as callbacks will just go to the executor in that case.

## Open issues (if applicable)

N/A. This has been provided in the library as an experimental
interface and used internally at Google for nearly 2 years.
