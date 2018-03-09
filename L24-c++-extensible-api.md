L24: C++ Extensible api
-----------------------
* Author(s): makdharma
* Approver: vjpai
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/14517
* Last updated: Feb 26, 2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal will enable extending functionality of GRPC C++ API by removing
the `final` keyword and making some class methods `virtual`. It will allow
alternate implementations of the API by that subclass.

## Background

Most GRPC C++ API classes are currently marked `final`. The rationale is that
having virtual methods might have adverse memory size and performance impact.
Additionally, adding `final` to public API classes later on would break existing
implementations that use classes derived from public API. Hence it is better and
safer to mark classes `final` unless there's a reason to make them extensible.

### Related Proposals:

## Proposal

* Remove `final` keyword from all public API classes.
* Move a subset of current private methods to protected. See the implementation
  PR for currently defined subset.
* Add protected getters/setters for a subset of private member variables. See
  the implementation PR for currently defined getters/setters.
* Mark core methods to `virtual`, so the functionality can be extended.
* Each such change should go through thorough performance evaluation and should not
  be accepted if there is any observed performance degradation.
* Do this work in stages. Begin with Server, ServerBuilder, and CompletionQueue
  classes, and extend to other classes as needed.


## Rationale

The original rationale for keeping methods private and classes non-extensible
has not held true. See https://github.com/grpc/grpc/pull/14359 for details of
performance eval in the extreme case. Meanwhile the `final` classes and
non-virtual methods preclude any experimentation with implementation. This
proposal will allow extending and experimenting with different server, client,
and support structure implementations.

The performance impact of removing `final` from all public
API classes was not measurable. Impact of adding `virtual` was noticable in
case of Inproc transport microbenchmarks and only for 256K byte messages.
However, this is a extreme case where virtual was added everywhere, which is not
the intention behind this proposal. The more realistic case is seen in
performance results of PR https://github.com/grpc/grpc/pull/14517, which shows
the impact of both adding `virtual` and removing `final` for a few
important classes is negligible.


## Implementation

The proposed implementation is in PR https://github.com/grpc/grpc/pull/14517.
It is limited in scope. It doesn't change every public class and method, but it
gives enough extensibility to experiment with alternate implementations.

## Open issues (if applicable)
