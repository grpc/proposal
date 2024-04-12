API changes: Completion Queue creation
----
* Author(s): Sree Kuchibhotla
* Approver: ctiller
* Status: Approved
* Implemented in: C/C++
* Last updated: March 21st, 2017
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/PNHNTuFQZaA

## Abstract
The proposal is to refine the completion queue creation API so that it provides better error-checking and makes it easier to implement new scenarios in future

## Background
1. A completion queue is created by using the following API
   - ` grpc_completion_queue *grpc_completion_queue_create(void *reserved)`

2. To get the result out of a completion queue, the caller must call one of the following two APIs:
``` C
   grpc_event grpc_completion_queue_next(
        grpc_completion_queue *cq,
        gpr_timespec deadline,
        void *reserved);

   grpc_event grpc_completion_queue_pluck(
        grpc_completion_queue *cq,
        void * tag,
        gpr_timespec deadline,
        void *reserved)
```

   `grpc_completion_queue_next` and `grpc_completion_queue_pluck` cannot both be called on the same completion queue. Doing so might cause the `grpc_completion_queue_pluck` call to be stuck indefinitely because the `tag` might have been returned to the caller via the `grpc_completion_queue_next` call. Currently we do not have a good way to prevent the user from doing this.

3. Internally, every completion queue currently has an associated _pollset_ containing file descriptors used for I/O.  gRPC core library does not create any threads on its own and instead all the rpc-related I/O (i.e polling the file descriptors, reading and writing messages) happens in the threads that call either `grpc_completion_queue_next` or `grpc_completion_queue_pluck`

4. A completion queue that can be used to listen to incoming RPCs or to be notified of server shutdown MUST be registered as a 'server completion queue' using one of the following two APIs:
``` C
   grpc_server_register_completion_queue()
   grpc_server_register_non_listening_completion_queue()
```
   Unless registered as a non-listening completion queue, a server completion queue will have the _listening fd_ (i.e the fd corresponding to the port on which the server is listening) in its _pollset_.

5. At the C++ layer, a grpc _Server_ may contain multiple _Services_. Each _Service_ may be either synchronous or asynchronous. If the Server contains atleast one synchronous service, the gRPC C++ library creates a few server completion queues and a thread pool to "poll" the completion queues for incoming RPCs

### Problems with the current Completion Queue APIs

- No error checks / validations to prevent the user from calling completion queue next and pluck APIs

- The API is not flexible enough to create new types of completion queue models in future; like for example, a call-back based completion queue, a single-threaded completion queue or a queue with a fixed number of threads) etc

- Tight coupling of completion-queue and pollsets presents some tricky challenges. If an application creates a completion queue, our current model dictates that the application should call `grpc_completion_queue_next` or `grpc_completion_queue_pluck` very regularly (preferably in a tight loop) to ensure progress happens.  This can lead to many gotchas like the following:
    - In case of a C++ server containing a _hybrid service_ (i.e a service where some methods are exposed using synchronous API and some with asynchronous API), it is important to ensure that `grpc_completion_queue_next` or `grpc_completion_queue_pluck` are actively called on all server completion queues that are created. (Note: The non-listening completion queue was created to fix this but it is still a bit hard to use this correctly)

   - In case of a _hybrid server_ (i.e a server implementing multiple _services_ some of which are synchronous and some asynchronous), we might end up with too many completion queues. This is because the presence of a synchronous service means that the gRPC C++ library already creates several server completion queues and a thread pool to poll them. The presence of asynchronous services means that the application server (using the gRPC c++ library) is potentially creating one or more server completion queues too and is actively polling them.  Too many threads polling the completion queues can potentially result in multiple threads polling the same set of fds in the underlying pollsets. This results in more contention at the completion queue / polling engine level (The exact details on why this happens is beyond the scope of this gRFC. See gRPC epoll based polling engine design [here](https://github.com/grpc/grpc/blob/master/doc/epoll-polling-engine.md) if interested - it is not required to read this for this gRFC)

### Related Proposals:  n/a

## Proposal

The proposal is to modify the current APIs to clearly specify what type of completion queue is being created/used.  To this end, we are proposing the following changes:

#### Changes to GRPC C Core API

##### 1. Create two enums: `grpc_cq_completion_type` and `grpc_cq_polling_type` at gRPC C core API layer

``` C
typedef enum {
  /* Events are popped out of the completion queue
     by calling grpc_completion_queue_next() API ONLY */
  GRPC_CQ_NEXT = 0,

  /* Events are popped out of the completion queue
     by calling grpc_completion_queue_pluck() API ONLY */
  GRPC_CQ_PLUCK

  /* In future we will add more types like GRPC_CQ_CALLBACK */
} grpc_cq_completion_type


typedef enum {
  /* Default option.  Completion queues will have an
     associated pollset and it is expected that
     grpc_completion_queue_next(), grpc_completion_queue_pluck()
     (or any other future polling APIs we might introduce) are
     called very actively to ensure I/O progress */
  GRPC_CQ_DEFAULT_POLLING,

  /* Similar to GRPC_CQ_DEFAULT_POLLING except that the pollset
     associated with the completion queue will not have any
     listening fds (if the completion queue is used as a server
     completion queue) */
  GRPC_CQ_NON_LISTENING,

  /* The completion queue will NOT have any associated
     pollset. This means, while grpc_completion_queue_next()
     or grpc_completion_queue_pluck() should be called to
     pop events out of the queue, it is not required to call
     them actively to ensure I/O progress */
  GRPC_CQ_NON_POLLING,
} grpc_cq_polling_type

```

##### 2. Introduce the concept of `grpc_completion_queue_factory` and add a factory-lookup API

``` C
/* The grpc_completion_queue_factory structure definition will remain internal
   and not exposed to the user */
typedef struct grpc_completion_queue_factory grpc_completion_queue_factory;

#define GRPC_CQ_CURRENT_VERSION 1
struct grpc_completion_queue_attributes {
  /* We expect to add more fields to this structure in future. So is important
     to pass a version number. */
  int version;  /* Initially this will be 1 */

  grpc_cq_completion_type cq_type;

  grpc_cq_polling_type cq_polling_type;
};

GRPCAPI const grpc_completion_queue_factory* grpc_completion_queue_factory_lookup(
    const grpc_completion_queue_attributes* attributes);
```

###### Internal structures (**not** exposed to users)

``` C
struct grpc_completion_queue_factory_vtable {
  grpc_completion_queue* (*create)(
      const grpc_completion_queue_factory *,
      const grpc_completion_queue_attributes *);
};

struct grpc_completion_queue_factory {
  char* name;
  void* data; /* Factory specific data */
  grpc_completion_queue_factory_vtable vtable;
};

```

##### 3.  Change the `grpc_completion_queue_create()` API to the following:

``` C
/** Create a completion queue. 
    factory: Completion queue factory obtained using grpc_completion_queue_factory_lookup API
    attr: Completion queue attributes. 
          Important: You MUST pass the same attributes you used to lookup the factory.
*/
GRPCAPI grpc_completion_queue *grpc_completion_queue_create(
    const grpc_completion_queue_factory *factory,
    const grpc_completion_queue_attributes *attr,
    void *reserved);
```

##### 4. Add the following two helper APIs for common cases:
``` C

/** This is equivalent to calling the following:
   grpc_completion_queue_attributes attr = { 1, GRPC_CQ_NEXT, GRPC_CQ_DEFAULT_POLLING };
   const grpc_completion_queue_factory* factory = grpc_completion_queue_factory_lookup(attr);
   grpc_completion_queue_create(factory, attr);
*/
GRPCAPI grpc_completion_queue *grpc_completion_queue_create_for_next(
    void *reserved); /* reserved MUST be NULL */

/** This is equivalent to calling the following:
   grpc_completion_queue_attributes attr = {1, GRPC_CQ_PLUCK, GRPC_CQ_DEFAULT_POLLING };
   const grpc_completion_queue_factory* factory = grpc_completion_queue_factory_lookup(attr);
   grpc_completion_queue_create(factory, attr);
*/
GRPCAPI grpc_completion_queue *grpc_completion_queue_create_for_pluck(
    void *reserved); /* reserved MUST be NULL */

```

##### 5.  Add validations to `grpc_completion_queue_next()` and `grpc_completion_queue_pluck()` APIs

Assert if `grpc_completion_queue_next()` is called on a GRPC_CQ_PLUCK type completion queue and vice versa

#### Changes to GRPC C++ API

C++ provides a `CompletionQueue` type class which has two methods `Next()` and `Pluck()` but only `Next()` is public.

As a part of this the following changes would be made:
- Add a private constructor to `CompletionQueue` that takes the completion type and the polling type.
- Make the public constructor to call the private constructor with  `GRPC_CQ_NEXT` and `GRPC_CQ_DEFAULT_POLLING`
- Change any internal code that uses `CompletionQueue::Pluck()` to use the private constructor when creating the `CompletionQueue` object

  ``` C++
  class CompletionQueue :.. {
  public:
    CompletionQueue() : CompletionQueue(GRPC_CQ_NEXT, GRPC_CQ_DEFAULT_POLLING) {}
  ..
  private:
    CompletionQueue(grpc_cq_completion_type completion_type, grpc_cq_polling_type polling_type) {
      grpc_completion_queue_attributes attr = { 1, completion_type, polling_type };
      cq_ = grpc_completion_queue_create(
                grpc_completion_queue_factory_lookup(&attr),
                &attr,
                NULL);
    }
    ..

    grpc_completion_queue* cq_;
    ..
  }

  ```

NOTE: There is NO need to add validations to the following APIs (to check the API call with the completion type) since it is already added at the gRPC C Core API level.

``` C++
  CompletionQueue::Next()
  CompletionQueue::AsyncNext()
```

#### Changes to all other wrapped language APIs
Similar to the one I described in C++ API changes section (to use the new `grpc_completion_queue_create_for_next` or `grpc_completion_queue_create_for_pluck` API)

## Rationale
The rationale for these changes is already discussed in the "Problems with the current Completion Queue APIs" section above.
To summarize, with these new changes:
  - It is relatively easier to write and reason-about complex servers (i.e servers containing a mix of sync/async services, hybrid services etc)
  - We get better validation/error-checks in the API and more flexibility in the API


## Implementation
All the relevant details are described in the proposal section above

## Open issues (if applicable)
- Should we even have a non-listening type.  The reason it was added (i.e Hybrid server scenario) can now be satisfied with a non-polling completion queue
  --Answer: Deprecate the grpc_create_non_listening_completion_queue() API

