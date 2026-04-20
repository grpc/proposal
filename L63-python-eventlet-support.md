Title
----
* Author(s): Akrog
* Approver: a11r
* Status: Draft
* Implemented in: python
* Last updated: 2/21/2020
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add an experimental custom IO manager to provide compatibility between the gRPC
and Eventlet libraries.

## Background

Eventlet is a concurrent networking library for Python in the same line as
Gevent.

Using epoll/libevent together with the greenthread library for coroutines
allows a blocking style of programming that is similar to threading, but
providing the benefits of non-blocking I/O.

The event dispatch is implicit and uses a greenthread, which means that when we
try to use the gRPC library with a Python application that uses Eventlets the
application will be blocked, because the eventlet library expects the Python
application to yield on the next I/O operation, yet the gRPC library won't,
because it will be blocked on the poll operation.

This means that no Python application that uses eventlets can use gRPC.  This
includes but is not limited to:

- Using the etcd3 python library with eventlets

- [OpenStack's Cinder](https://docs.openstack.org/cinder/latest/) using TooZ
  with the etcd3 driver as a DLM

- Using [cinderlib](https://docs.openstack.org/cinderlib/latest/) together with
  gRPC to:

  - An efficient imlementation of a
    [CSI](https://github.com/container-storage-interface/spec) plugin such as
    [Ember-CSI](https://ember-csi.io)

  - oVirt using a gRPC service for their [Managed Block Storage feature](
    https://www.ovirt.org/develop/release-management/features/storage/cinderlib-integration.html)

  - [Swift object storage switched to plain HTTP from gRPC](
    https://opendev.org/openstack/swift/commit/21d710b1f69b0e8e4e816d4c94c3ebb804456fd5?lang=sv-SE)

### Related Proposals:

N/A

## Proposal

This gRFC proposes that we add an experimental custom I/O manager like the one
we currently have for gevent.

Eventlet and gevent use greenthreads differently so their implementation will
not be exactly the same.

The implementation will require:

- Schedule asynchronous calls on greenlets so they can do the callbacks when
  they complete their execution.

- Make sure we schedule asynchronous calls on the right eventlet hub, the one
  from the main thread where we do the polling.

- A custom Event, because eventlet's cannot work between different native
  threads, and we will get kicks from the main thread as well as the one with
  the ThreadPoolExecutor.

- Use a socket to wake the main thread hub so we don't have to wait until it
  timeouts.

- Signal the greenthread that is waiting on the accept socket call when we
  close the socket so the server can gracefully stop.

## Rationale

### Alternative Design: Native threads

The blocking of the polling can resolved if we use native threads for the
polling methods that would block.

This can be done using eventlet's `tpool.Proxy` class on the server and
`completion_queue` and then passing removing the proxy when calling C code.

This is how [Ember-CSI is currently solving the issue](
https://github.com/embercsi/ember-csi/blob/148cc4a09db22df8aa4772dfb3696160cf37fe28/ember_csi/workarounds.py#L25-L54).

The advantage of this approach is that the amount of code is small, but the
downside is greater:

- Server won't stop gracefully: Eventlet does not signal socket readers/writers
  when the socket is closed from another greenthread, so the accept call that
  will be awaiting won't know that the socket has been closed and will never
  reach the point where it makes the accept callback with an error, thus
  blocking the graceful stop of the gRPC server.

- Performance is not good.  In tests the I/O manager has shown up to 4x better
  performance.

## Implementation

PR: https://github.com/grpc/grpc/pull/22062
