## Abstract

Set clear expectations of the gRPC fork support and provide implementation details.

## Background

The `fork` Posix API creates a new process, which is a duplicate of the parent
process. The new process receives a complete copy of the parent process's memory
and begins execution at the same point. The `fork` call returns `0` in the child
process and the child's non-zero PID in the parent process.

More details on fork API can be found in your OS manual or [online][fork-man].

Some language runtimes, such as Python, rely on fork API to better utilize modern
hardware, such as multi-core CPUs. Python runtime has a [global interpreter lock][gil]
that limits multi-threaded performance and relying on fork API allows to bypass
this restriction.

### Issues with the fork API

The child process created after a fork call is not a complete copy. Possibly
the most notable omission is that only the thread where `fork` was called would
be recreated in the child process. Following potential issues may be caused
by this:

- Locks held by those threads will never be released, resulting in the potential 
deadlocks.
- Work performed by those threads will never be finished.
- There is a discrepancy between the state in internal gRPC structure and actual
system state (e.g. memory pool may expect it has a certain number of threads while
none of those are running).

While threads are not properly copied, all file descriptors are duplicated in both
child and parent processes. This causes issues when multiple processes are trying
to access the same file descriptors, e.g.:

- Reading and writing to the same FD from multiple processes can cause data loss
or data corruption.
- Using same pipe or event FD from multiple process may result in a wrong process
waking up and another process getting stuck

[fork-man]: https://man7.org/linux/man-pages/man2/fork.2.html
[gil]: https://en.wikipedia.org/wiki/Global_interpreter_lock

## Proposal

### Proposed Behavior

1. The parent process will retain its state, including open file descriptors.
2. There may be recoverable errors reported in the parent process immediately
post-fork.
3. All file descriptors must be explicitly reopened in the child process.
Internal file descriptors, like pollers, event FDs and such will be reopened
automatically.
4. gRPC will remain in correct and consistent state in both parent and child
Processes. Both processes can fully use gRPC, such as open new connections or
start new services.

### Stopping Background Threads

All gRPC working threads need to be stopped during the fork and restarted
post-fork. This includes suspending i/o, polling and timer that all rely on
background threads.

Sequence of operations:

1. Prefork. Only one process exists at this point.
    - Stop polling open file descriptors. This ensures that no IO happens during
      the fork.
    - Stop the timer manager and save all tasks.
    - Shutdown thread pool to stop all the threads.
    - Notify generic fork listeners that fork is about to happen. Some gRPC
      facilities may decide to perform special fork handling and they can do it
      by adding a fork listener to poller.
2. Post-fork parent. This restarts facilities stopped above.
    - Restart thread pool
    - Restart timers
    - Notify listeners
    - Restart polling

    At this point parent process resumes all IO, possibly reporting spurious
    errors and/or timeout triggered.
3. Post-fork child. Closes file descriptors and restores gRPC to valid state.
    - Close open file descriptors and advance file descriptor "generation" (see
    below for details)
    - Restart thread pool
    - Restart timers
    - Notify listeners
    - Restart polling

#### Undesired gRPC recovery post-fork

The child process will inherit the gRPC state from its parent. That means that
gRPC may attempt to recover from errors as the timers fire and other events
happen. E.g. listeners may start reporting errors or connections will
be reestablished.

_It is considered a client responsibility to shutdown all gRPC facilities in
the child process. It is advised to perform fork before any gRPC services are
started and/or connections are established._

Alternative considered: requiring the gRPC to be explicitly restarted post-fork
in the child process, e.g. by a call to `grpc_init`. This approach will further
complicate the fork process in most cases and is error-prone, as well as
increasing overall impact of fork support on gRPC implementation.
      
### Handling File Descriptors

The child process should not be able to access file descriptors opened before
the fork. Parent process should still be able to access file descriptor after
the fork process completes.

For type safety and better tooling support, we will replace all integer file
descriptor with an instance of FileDescriptor class.

File descriptor class will be a wrapper around integer FD and can be annotated
with `__attribute__((trivial_abi))` to remove overhead when passing it by value.
 There is no need to keep negative values, with `0` representing the broken FD.

#### Close all open file descriptors

Open file descriptors often require kernel to allocate resources, such as memory
for buffers and bookkeeping. Child process needs to close all gRPC file
descriptors that were open pre-fork.

For fork support, each Posix poller implementation provides a file descriptor
factory that wraps Posix APIs like `socket`, `epoll_create1` and similar. It will
also maintain a collection of open file descriptors that is necessary to close all
the descriptor.

Closing file descriptors in the course of normal operation will also go through
the service.

Initial implementation will use mutex for maintaining the collection
(opening/closing the FDs). Collection can be compiled out when the gRPC is built
 without the fork support. Note that this mutex will only be required when either:

- File descriptor is opened (e.g. a new connection is opened, poller is created)
- File descriptor is closed (connection is closed, poller shutdown).

#### Preventing gRPC objects from accessing file descriptors that were closed pre-fork

Many objects through the Event Engine and sometimes even outside it hold the file
descriptor values. This proposal will not alter this behavior to avoid introducing
new overhead or extensive code changes.

This results in a potential for those objects accessing wrong object because Posix
reuses previously closed file descriptors. E.g. endpoint may need to access fd #5
that was previously a socket connection but is now representing a pipe connection
used for wakeup.

To prevent this we will:
1. Annotate each file descriptor with a "generation", starting with `1` when
the initial process is started.
2. Track generation in each poller. Generation is incremented post-fork in
the child process.
3. Wrap Posix calls and check that the file descriptor generation matches
the current generation. Generation mismatch will be reported as an i/o error
and will allow gRPC to handle the failures as other i/o failures, such as
reopen a connection.

#### Generation tracking

Generation can be an atomic, tracked in the file descriptors service. Posix uses
lowest file descriptor number available so we can reserve top 4 bits for 
the generation, allowing the application to use up to 2^28 file descriptors.
The generation counter is reset after 8-16 generations.

Generation check can be performed like this:

```c++
uint32_t fd = grpc_fd.fd();
uint32_t generation = poller_generation.load(std::memory_order_relaxed) & 0xf;
bool is_right_generation = (fd >> 28) == generation;
```

Note that checking the generation does not require holding the mutex.

Note that generation increment only happens when all i/o is stopped, which may
enable us to perform additional optimizations.

#### Wrapping Posix system calls

It was discussed offline, that we may want to consolidate Posix calls in one
"service" class for purposes like fuzz testing. This proposal includes
implementing this service.

This service will check the generation as discussed above, and then either
return error or perform the system call, returning possible Posix error to the
callee.

For efficiency, the service will not be a 1:1 mapping of Posix APIs. It will
combine some calls that usually happen together (such as opening an FD and
making it non-blocking) to reduce the API surface.

#### Reporting operation result

Most Posix calls return a single integer, where zero value usually indicates
success (or special state, like EOF is a read call), positive value is more
data (e.g. number of bytes read or fd number). Negative values either contain
specific errors or indicate the need to consult `errno` global variable to
get error details.

Some of the calls will now need to return new file descriptors and a new kind
of error (wrong file descriptor generation) will need to be incorporated.
This will be implemented as a new structure:

```c++
struct PosixResult {
    enum class Kind {
        kOk;              // Operation does not return a file descriptor and
                          // return value was >= 0. native_result holds the
                          // original return value.
        kFileDescriptor;  // fd field contains a new ready file descriptor
        kError;           // Check native_result and errno for details
        kWrongGeneration; // System call was not performed because file
                          // descriptor belongs to the wrong generation.
    };
    Kind kind;
    // Raw value
    int native_result;
    // gRPC wrapped FileDescriptor, as described above
    FileDescriptor fd;
    // errno value on call completion, in order to reduce the race conditions
    // from relying on global variable.
    int errno;
}
```

Alternatives considered: An implementation was made that relies on `absl::Status`
and `absl::StatusOr`. It was decided that returning these objects would result
in unnecessary memory overhead caused by the objects size.