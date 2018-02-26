Title
----
* Author(s): makdharma
* Approver: vpai
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/14517
* Last updated: Feb 26, 2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal will enable extending functionality of GRPC C++ API.

## Background

All GRPC C++ API classes are currently marked "final". The rationale is that
having  virtual methods might have adverse memory size and performance impact.
Additionally, adding "final" to public API classes later on would break existing
implementations that use classes derived from public API. Hence it is better and
safer to mark classes "final" unless there's a reason to make them extensible.

### Related Proposals:

## Proposal

* Remove "final" keyword from all public API classes.
* Move current private core methods and member variables to protected.
* Mark core methods to virtual, so the functionality can be extended.
* Each such change should go through thorough performance evaluation.
* Do this work in stages. Begin with Server, ServerBuilder, CompletionQueue
  classes and extend to other classes as needed.


## Rationale

The original rationale for keeping methods private and classes non-extensible
has not held true. The performance impact of removing "final" from all public
API classes was not measurable. Meanwhile the final classes and non-virtual
methods preclude any experimentation with implementation. This proposal will
allow extending and experimenting with different server and client
implementations.

## Implementation

## Open issues (if applicable)
