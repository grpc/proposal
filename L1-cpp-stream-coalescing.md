# C++ streaming coalescing API's

* Author(s): ctiller
* Approver: a11r
* Status: Draft
* Implemented in: C++
* Last updated: 2017/01/13
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/GfxB7nuV9GY

## Abstract

Provide C++ API's to coalesce:
- initial metadata and first streaming message
- final streaming message and trailing metadata
into the same core batch.

## Background

Currently when making a streaming call using gRPC and sending a single message,
three core batches are created: one for initial metadata, one for the message
itself, and one for trailing metadata.

Since the underlying transport has no knowledge that future messages are coming,
it's forced to assume it should initiate writes immediately for each of them.
In typical cases, this results in three sendmsg syscalls per stream.

The current API allows some coalescing via WriteOptions. One can do:
```c++
auto s = stub->Foo(...);
s->Write(first_message, WriteOptions().set_buffer_hint());
s->WritesDone();
```
This is difficult to discover, and further more it performs two round-trips
through the C Core stack, where it would be possible to do one.

### Related Proposals:
N/A

## Proposal

### Expand WriteOptions to include an end-of-stream bit
```c++
class WriteOptions {
 public:
  // ... existing interface ...

  // corked bit: aliases set_buffer_hint currently, with the intent that
  // set_buffer_hint will be removed in the future
  WriteOptions& set_corked();
  WriteOptions& clear_corked();
  bool is_corked();

  // last-message bit: indicates this is the last message in a stream
  // client-side:  makes Write the equivalent of performing Write, WritesDone in
  //               a single step
  // server-side:  hold the Write until the service handler returns (sync api)
  //               or until Finish is called (async api)
  WriteOptions& set_last_message();
  WriteOptions& clear_last_message();
  bool is_last_message() const;
};
```

### Add a convenience WriteLast method to ClientWriterInterface, ClientReaderWriterInterface, ServerWriterInterface, ServerReaderWriterInterface
```c++
// Perform Write, WritesDone in a single step
void WriteLast(const W& msg, WriteOptions options) {
  Write(msg, options.set_last_message());
}
```

### Add a convenience WriteLast method to ClientAsyncWriterInterface, ClientAsyncReaderWriterInterface, ServerAsyncWriterInterface, ServerAsyncReaderWriterInterface
```c++
// Perform Write, WritesDone in a single step
void WriteLast(const W& msg, WriteOptions options, void* tag) {
  Write(msg, options.set_last_message(), tag);
}
```

This will require finally exposing WriteOptions to async code also:
```c++
void Write(const W& msg, WriteOptions options, void* tag);
```

### Add a WriteAndFinish method to ServerAsyncWriterInterface, ServerAsyncReaderWriterInterface
```c++
// Perform Write, Finish in a single step
void WriteAndFinish(const W& msg, WriteOptions options, const Status& status void* tag);
```

### Expand ClientContext to allow corking metadata
```c++
class ClientContext {
 public:
  // ...

  // flag that metadata should be corked (and not sent until the first message
  // is sent
  void set_initial_metadata_corked(bool corked);
};
```

## Rationale

The last-message bit provides an avenue for our C++ wrapper to form a C core
batch that contains both a send message and a half close.

The WriteLast and WriteAndFinish methods, in and additional Stub stream
constructors provide first class discoverability to these API's. Importantly
code completion tools should offer them as suggestions to new developers, and
they'll appear in top-level documentation.

The ClientContext change allows a C++ implementation that forms a C core batch
containing both initial metadata and the first message.

By combining a corked initial metadata and WriteLast, clients can coalesce all
of initial metadata, only message send, and half close.

## Implementation

This should be straightforwardly implementable in the C++ layer.
