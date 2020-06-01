EventManager API
----
* Author(s): @muxi
* Approver: @markdroth
* Status: Draft
* Implemented in: gRPC Core
* Last updated: June 1, 2020
* Discussion at: <google group thread> (filled after thread exists)

## Abstract
This proposal describes a plan to upgrade gRPC's event engine to a newer set of interfaces called EventManager API.The gRPC team is making a fundamental change in the gRPC core's polling model. Instead of borrowing threads from the application and giving the application control over which threads poll which fds, gRPC will migrate to a new model based on EventManager, which provides dedicated polling threads with an internal thread pool. User callbacks and timers will also be handled by the EventManager. The EventManager interface will be a public API, which allows users to integrate gRPC into the event loops of their applications if necessary.

## Background
The gRPC has been using a custom event handling framework called iomgr API. The original iomgr API had only two implementations, one for posix (Linux and MacOS) and one for Windows. As part of an effort to improve performance, we changed the posix iomgr implementation to support pluggable polling engines, selectable at run-time.  We then wrote [a whole bunch of different polling engines](https://github.com/grpc/grpc/blob/master/doc/core/grpc-polling-engines.md) to experiment on and determine which one(s) worked best.  Some of them didn't work out and have been deleted, but we still support three of them (see below).

The EventManager interface aims at simplifying the usage and maintenance of gRPC event framework. It allows users of gRPC to write their own event management implementation to be used by gRPC for specific needs. The project also paves the way for supporting features such as the C++ callback API in gRPC (See proposal [gRFC L67: C++ callback-based asynchronous API](https://github.com/grpc/proposal/pull/180)).

## Proposal
The following event manager interfaces will be available in the gRPC core.

```cpp
// An external handle to an EventManager task, which can be used to cancel the 
// task. The EventManager task is added via RunFunction.
struct TaskKey;

// An external handle to an EventManager alarm, which can be used to cancel the
// alarm. The EventManager alarm is added via AddAlarmIn.
struct AlarmKey;

// Scheduling priority for closures and alarms added into the EventManager.
enum Priority {
  PRIORITY_MIN = 0,
  PRIORITY_LOW = 0,
  PRIORITY_MED = 1,
  PRIORITY_HIGH = 2,
  PRIORITY_IMMEDIATE = 3,
  PRIORITY_SYSTEM = 4,
  PRIORITY_MAX = 4,
  NUM_PRIORITIES = 5
};

// A top-level base class of EventManagerInterface.
class BaseEventManagerInterface {
 public:
  virtual ~BaseEventManagerInterface();

  // Adds a new one-shot alarm, which will be scheduled at the deadline to run the 
  // closure with true. The deadline here is an absolute time. The AlarmKey pointer 
  // is an output parameter which can be used to cancel the alarm before it fires. 
  // Note that the closure should not be blocking or long-running, as it would 
  // block the worker thread in the EventManager from polling the file 
  // descriptors. In case that the alarm is cancelled successfully in 
  // ConsistentCancel, the closure will be run with false in the calling thread of 
  // ConsistentCancel. Callers of ConsistentCancel need to be cautious to avoid 
  // deadlock.
  virtual void ScheduleAlarmIn(gpr_timespec deadline, std::function<void(bool)> c, Priority priority, AlarmKey* key = nullptr) = 0;

  // Adds a closure to the EventManager, which will be run at the earliest possible 
  // time. The TaskKey pointer is an output parameter which can be used to cancel 
  // the closure at a later time (currently not supported in 
  // BaseEventManagerInterface). Note that the closure should not be blocking or 
  // long-running, as it would block the worker thread in the EventManager from 
  // polling the file descriptors.
  virtual void RunFunction(std::function<void()> handler, Priority priority, TaskKey* key = nullptr) = 0;

  struct Task {
    std::function<void()> handler;
    Priority priority;
    TaskKey* key;
    int internal_added_queue;
  };
  // Adds multiple closures to the event management system all at once.
  virtual void RunFunctions(Task* tasks, int length) = 0;

  // Cancels an alarm which has been added to the EventManager via 
  // ScheduleAlarmIn(). This function will run the closure with false in the 
  // calling thread if it successfully cancels the alarm.
  virtual void ConsistentCancel(const AlarmKey& key) = 0;

  // Cancels an alarm as ConsistentCancel(), but wait if the task is currently 
  // running.
  virtual void BlockingConsistentCancel(const AlarmKey& key) = 0;

  // These two methods adjust a refcount of the EventManager, preventing it from 
  // shutdown if the refcount is positive. ShutdownRef() should be called before 
  // adding the first closure to the EventManager, and ShutdownUnref() can be 
  // called after removing the last closure from the EventManager. Also, if you 
  // need to keep the EventManager serving before receiving an RPC response, you 
  // need to call ShutdownRef() before sending the RPC, and invoke ShutdownUnref() 
  // when the response arrives.
  virtual void ShutdownRef() = 0;
  virtual void ShutdownUnref() = 0;
};
```

The BaseEventManagerInterface is platform agnostic in order to accommodate the different polling models on different platforms (e.g. POSIX vs Windows).

A specific interface EpollEventManagerInterface is derived from BaseEventManagerInterface for all the epoll-based polling models.

```cpp
// Represents a file descriptor on epoll-capable platforms.
class EpollDescriptorInterface {
 public:
  virtual ~EpollDescriptorInterface();

  // Returns the underlying file descriptor.
  virtual int fd() const = 0;

  // Returns the EventManager object where this Descriptor is registered.
  virtual EpollEventManagerInterface* event_manager() const = 0;

  // Executes all the registered notification closures, and closes the underlying 
  // file descriptor.
  virtual void Close() = 0;

  // Returns whether the underlying file descriptor is closed or not.
  virtual bool IsClosed() const = 0;

  // Dissociates the underlying file descriptor from the Descriptor object, and 
  // stop monitoring the file descriptor. Executes all the notification closures 
  // registered for this file descriptor, and returns the file descriptor.
  virtual int ReleaseFD() = 0;

  // Sets various priorities for this file descriptor. The priorities are ignored 
  // if the Descriptor does not support Priority.
  virtual void SetReadPriority(Priority priority) = 0;
  virtual void SetWritePriority(Priority priority) = 0;
  virtual void SetErrorPriority(Priority priority) = 0;

  // Registers a closure to be run immediately in a worker thread of the 
  // EventManager, when the file descriptor becomes readable/writable/has error.
  virtual void NotifyWhenReadable(std::function<void()> upcall) = 0;
  virtual void NotifyWhenWritable(std::function<void()> upcall) = 0;
  virtual void NotifyWhenHasError(std::function<void()> upcall) = 0;

  enum NotificationOutcome {
    NOTIFY_FD_BECAME_READY,
    NOTIFY_TIMED_OUT
  };

  // Similar to NotifyWhen*, but with a deadline. If the file descriptor state 
  // changes before the deadline, then upcall will be called with 
  // NOTIFY_FD_BECAME_READY. If the deadline expires before the descriptor state 
  // changes, then upcall will be called with NOTIFY_TIMED_OUT.
  virtual void NotifyWhenReadableOrTimeout(std::function<void(NotificationOutcome)> upcall, gpr_timespec deadline) = 0;
  virtual void NotifyWhenWritableOrTimeout(std::function<void(NotificationOutcome)> upcall, gpr_timespec deadline) = 0;

  // Clears the descriptor's status. This allows the NotifyWhen* calls to 
  // distinguish between old and new events.
  virtual void ClearReadable() = 0;
  virtual void ClearWritable() = 0;
  virtual void ClearHasError() = 0;

  // Forces the descriptor’s status to change, and runs the closures if registered. 
  // This is especially useful for tearing down closures associated with this 
  // descriptor by forcing them to run without waiting for the event on the 
  // underlying socket.
  virtual void ForceReadable() = 0;
  virtual void ForceWritable() = 0;
  virtual void ForceHasError() = 0;

  // Returns a human-readable description of the file descriptor state.
  virtual string ToString() const = 0;
};

class EpollEventManagerInterface: public BaseEventManagerInterface {
 public:
  virtual ~EpollEventManagerInterface();

  virtual EpollDescriptorInterface* RegisterFileDescriptor(int fd) = 0;

  virtual void DeleteDescriptor(EpollDescriptorInterface* d) = 0;

  // Adds a new one-shot alarm with a slack, which will be scheduled at the given 
  // deadline plus the timespan specified in slack. Reducing the precision of 
  // alarms through a timer slack allows the EventManager to perform 
  // timer-coalescing, which minimizes thread wake-up and has the potential of 
  // substantially reducing CPU utilization.
  virtual void ScheduleAlarmInWithSlack(gpr_timespec deadline, std::function<void(bool)> c, Priority priority, TaskKey* key, gpr_timespec slack) = 0;
};
```

Note that there will be no native implementation provided by gRPC for the epoll based interface in the first step. Instead, gRPC will provide a default event manager implementation based on libuv. The libuv EM implementation will be applied to all the gRPC languages based on gRPC core, and the users may choose to override it with their custom implementation of the EM interface. gRPC may release more native event manager implementations down the road map.
```cpp
// Represents a file descriptor when using libuv.
class LibuvDescriptorInterface {
 public:
  virtual ~LibuvDescriptorInterface();

  // Returns the underlying TCP or UDP handle. Valid until Close() is called
  // (i.e. not thread-safe with Close()).
  virtual uv_handle_t fd() const = 0;

  // Retruns the EventManager object where this descriptor is registered.
  virtual LibuvEventManagerInterface* event_manager() const = 0;

  // Executes all the registered notification closures, and closes the
  // underlying TCP or UDP handle.
  //
  // Note that we cannot release the underlying TCP or UDP handle, since it is
  // initialized with the libuv event loop of LibuvEventManager, and it is not
  // thread-safe to access the handle externally.
  virtual void Close() = 0;
  virtual bool IsClosed() const = 0;

  // Registers a closure \a upcall to be run immediately in a worker thread of
  // the LibuvEventManager, after libuv reads from the socket. \a alloc_cb will
  // be passed to libuv to allocate the read buffer. Note that this will start
  // a read asynchronously.
  //
  // It is an error to call NotifyAfterRead() again before the previous closure
  // completes.
  //
  // If there is more to read on the socket, re-arms a closure after upcall
  // finishes.
  virtuar void NotifyAfterRead(std::function<void(LibuvReadStatus)> upcall,
                       uv_alloc_cb alloc_cb) = 0;

  // Registers a closure \a upcall to be run immediately in a worker thread of
  // the LibuvEventManager, after libuv writes to the socket. \a info contains
  // the write message, and target address in cases of UDP. Note that this will
  // start a write asynchronously.
  //
  // It is an error to call NotifyAfterWrite() again before the previous closure
  // completes.
  //
  // If there is more to write, re-arms a closure after upcall finishes.
  virtual void NotifyAfterWrite(std::function<void(int)> upcall, LibuvWriteInfo info) = 0;

  // Accessor and mutator of arbitrary user-defined data. NOT thread-safe.
  virtual void* data() = 0;
  virtual void set_data(void* d) = 0;
};

class LibuvEventManagerInterface: public BaseEventManagerInterface {
 public:
  virtual ~LibuvEventManagerInterface();

  virtual LibuvDescriptorInterface* RegisterFileDescriptor(uv_handle_t* fd) = 0;

  virtual void DeleteDescriptor(LibuvDescriptorInterface* d) = 0;
};
```
Registration of an EventManager implementation with Core
User implementations of EventManager can be registered with gRPC Core by the following API:
```cpp
// Register a user-provided EventManager, which will be used globally in
// gRPC. gRPC won't take the ownership of this EventManager; the
//  EventManager must outlive gRPC.
//
// The function must be called before gRPC is initialized (i.e. grpc_init is first called).
// It can only be called at most once; otherwise an exception will be thrown. If the function 
// is not called at all, gRPC will use its internal default EventManager implementation.
void grpc_register_global_event_manager(BaseEventManagerInterface *em);
```

## Limitations
### Using C++ language
The EventManager API is a C++ interface provided by gRPC core. Any user of the feature (including the gRPC wrapping languages) that provide an alternative EventManager implementation needs to support C++. This may be a problem for some of the wrapping languages and users, but we expect the friction to be small. Particularly, any wrapping language that does not need to provide their own EventManager implementation or interact with an EventManager implementation will not be required to use C++.

### Link Limitations on Windows
On Windows platforms, the following limitations apply when linking gRPC:
* If a user module provides an EventManager implementation that gRPC should use, gRPC must be linked statically with the module.
* If a user module uses a gRPC internal EventManager implementation through the EventManager API (e.g. to run a function or schedule a timer), gRPC must be linked statically with the module.

## Rationale
### Problems with the current iomgr API
We identified a few problems with the current iomgr API:
* The current iomgr APIs are a mess; the semantics are extremely confusing, which makes our code very hard to maintain and reason about.  This complexity slows down our velocity on new features and makes it hard to debug and maintain existing code.
* The current iomgr APIs are extremely invasive and viral, requiring pollset\_sets to be plumbed into many different parts of the client\_channel code.
* There is a lot of inefficiency due to the number of levels of indirection in our current code.
* There are known bugs in several of our common polling engines, and we don't really have anyone left on the team who understands them well enough to try to fix them.
* From a strategic point of view, it does not make sense for us to spend time and effort to maintain multiple polling APIs.

To address these problems, we would like to get to a point where we eliminate the iomgr API in favor of simply using the EventManager API, and any new polling mechanism we need to support (windows, gevent, cfstream, etc) is handled via an EventManager implementation.

### Link Limitations on Windows
The limitation is due to a programming restriction on Windows. On windows platforms, [it is required that memory allocated inside a DLL must be freed inside the same DLL](https://devblogs.microsoft.com/oldnewthing/20161209-00/?p=94905). Since the EventManager interface is an extension interface, it is not unlikely that the EventManager implementation and gRPC live in different modules, e.g. gRPC lives in a DLL and the application providing EventManager implementation links to it. To avoid potential violation of the rule, there are two options: 1) design the API in a way that guarantees memory allocation and de-allocation do not happen on different sides of the API boundary, or 2) require the EventManager implementation to live in the same module as gRPC. As the EventManager API uses std::function to incorporate use cases of some users, Option 1 is not available. That is because std::function cannot take any deallocator with it, hence can carry memories across the API boundary to be de-allocated on the other side. That leaves us with Option 2 to require that whenever the EventManager interface is used, the code on the two sides must reside in the same module.

## Implementation
The EventManager interface and the libuv-based EventManager implementation will be completed by @guantaol.

