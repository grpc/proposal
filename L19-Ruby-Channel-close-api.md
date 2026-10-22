Ruby Client Stub close API
----
* Author(s): apolcyn
* Approver: nnoble,murgatroid99
* Status: In Review
* Implemented in: Ruby
* Last updated: 1/18/2018
* Discussion at: https://groups.google.com/forum/#!searchin/grpc-io/l19%7Csort:date/grpc-io/XmMA5Ex1HSQ/Q5am3z5bAgAJ

## Abstract

Provide a surface API to explicitly release all resources held by ruby client stub objects.

## Background

[An introduction of the necessary background and the problem being solved by the proposed change.]

Similar to the discussion in https://github.com/grpc/grpc/issues/12531, there
should be a `close` method for client stubs in Ruby to avoid reliance on the
garbage collector to release physical resources. There is currently a
`close` method on ruby `GRPC::Core::Channel`s, but when one uses the `GRPC::ClientStub.new`
API without providing it a channel, the channel is hidden from the surface and
then the only way to close the channel explicitly is with a hack
(`stub.instance_variable_get(:@ch).close`).

## Proposal

Provide an API to explicitly release resources that a client stub holds.
To avoid any possible collision with service methods on the stub, this API might look like:

```
GRPC::ClientStub.close(stub)
```

## Open issues (if applicable)

https://github.com/grpc/grpc/issues/12531
