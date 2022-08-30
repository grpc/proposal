API changes for `ClientAsyncResponseReaderInterface` destruction
----
* Author: jacobsa
* Approver: soheilhy
* Status: Draft
* Implemented in: C++
* Last updated: 2021-05-14
* Discussion at: https://groups.google.com/g/grpc-io/c/IzRZVDo4W8c

## Abstract

Currently although
[`grpc::ClientAsyncResponseReaderInterface`](https://github.com/grpc/grpc/blob/7e14d23ab46e1b0924d6c3e40797ed1d587aee7a/include/grpcpp/impl/codegen/async_unary_call.h#L37-L72)
has a virtual destructor, `std::default_delete` is specialized such that the
destructor never actually runs. So the only subclasses that can be implemeneted
without hacks are those like `grpc::ClientAsyncResponseReader`, which knows it
is always allocated on an arena and doesn't care about its destructor running.
This proposal provides a plan for guaranteed destructor invocation, such that
subclasses can count on their destructor running and their backing memory being
freed, as with any other C++ class.

## Background

We currently [specialize](https://github.com/grpc/grpc/blob/7e14d23ab46e1b0924d6c3e40797ed1d587aee7a/include/grpcpp/impl/codegen/async_unary_call.h#L404-L408)
`std::default_delete` for
`ClientAsyncResponseReaderInterface` to do nothing. This means that, despite the
fact that async stub methods return
`std::unique_ptr<ClientAsyncResponseReaderInterface>`, it's impossible to have a
subclass of that interface that is allocated on the heap or whose destructor has
side effects without a hack that manages ownership _outside of_ the `unique_ptr`
returned by the stub.

This was introduced by
[`dd36b153`](https://github.com/grpc/grpc/commit/dd36b15315cd691e86a94d4574bd9f3e3a33633f),
which added the specialization for `grpc::ClientAsyncResponseReader` itself, and
[`e8a61d63`](https://github.com/grpc/grpc/commit/e8a61d63b5a6db0f81b688481a9485d412e5a41e),
which did so for the interface. These commits didn't document their intent, but
I suspect it was a combination of the following facts:

*   In production uses, `ClientAsyncResponseReaderHelper::Create`
    [allocates](https://github.com/grpc/grpc/blob/7e14d23ab46e1b0924d6c3e40797ed1d587aee7a/include/grpcpp/impl/codegen/async_unary_call.h#L96-L99)
    the `ClientAsyncResponseReader` on an arena.

*   Although it's technically non-trivial, the `ClientAsyncResponseReader`
    destructor doesn't actually have any important side effects.

*   Therefore it's not necessary to run the destructor, and it's not correct to
    use `delete` to free the backing memory.

From this point of view, the original commit (`dd36b153`) is technically
correct: `std::unique_ptr<ClientAsyncResponseReader>` doesn't actually need to
do anything. But:

*   Of course the objects may be deleted other ways. If I have a
    `std::unique_ptr<ClientAsyncResponseReaderInterface>` (i.e. without a
    specific deleter type), it would be natural for me to assume that I can do
    `delete p.release()` to get the same effect as destroying the unique
    pointer. But that is not true, and doing so will be undefined behavior.

    So from this point of view `dd36b153` is not correct, or at least makes for
    a surprising API.

*   `ClientAsyncResponseReader` is not the only subclass of the interface. Even
    though subclasses can override the virtual destructor to add a side effect,
    their destructors do not actually run. This results in subtle bugs and
    confusing behavior, specially because `delete p.release()` works differently
    than just letting a `std::unique_ptr p` go out of scope.

    So from this point of view `e8a61d63` is also not correct.

This is not a theoretical concern: there are multiple hacks in user code working
around this with a grumbling tone. For example, see
[this hack](https://github.com/googleapis/google-cloud-cpp/blob/74de20227371119ea8b8c788d37ae4e24be8ed5d/google/cloud/bigtable/testing/mock_response_reader.h#L77-L106)
in the cloud bigtable library that uses the phrase "terrible, horrible, no
good, very bad hack" that involves managing a
`ClientAsyncResponseReaderInterface` object with **two** `unique_ptr`s (making
them not so unique).

## Proposal

The plan is to land this change a single commit that does the following:

*   Add a `virtual void Destroy()` method to
    `ClientAsyncResponseReaderInterface`, and update the `std::default_delete`
    specializations to call it.

*   Make the interface itself implement the method by doing `delete this`, a
    suitable default behavior that frees future subclasses from having to think
    about this problem at all.

*   Make `ClientAsyncResponseReader` implement the method by doing nothing,
    with an explanation that it knows it is allocated on an arena and has
    historically never run its destructor.

There is no behavior change for production code that uses
`ClientAsyncResponseReader` itself.

Other subclasses of `ClientAsyncResponseReaderInterface` will see a behavior
change. Their maintainers need to either override `Destroy` in the same manner
as `ClientAsyncResponseReader` to restore the former behavior, or remove the
hacks like the one described above to gain the usual ownership semantics of
`std::unique_ptr`. However the change is likely to only affect those mocking the
async API, which was never officially supported, so the commit will only bump
the minor version.

## Rationale

I came across this problem while trying to write mocking support for async
methods. It works just fine once this problem is fixed, but this problem makes
it extremely awkward otherwise.

The obvious alternative is to leave things as-is, but this requires awful hacks
like the one described above, making mocking difficult and dangerous. Since the
behavior change downsides primarily apply to tests, and the overall goal here
is to make tests significantly less hard to deal with, this seems worth it.
