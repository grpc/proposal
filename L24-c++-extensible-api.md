Title
----
* Author(s): makdharma
* Approver: a11r
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/14517
* Last updated: Feb 26, 2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal will enable extending functionality of GRPC C++ API.

## Background

All GRPC C++ API classes are currently marked "final". We would like to remove
this keyword so the API classes can be extended.

### Related Proposals:

## Proposal

* Remove "final" keyword from Server, ServerBuilder classes.
* Move some core methods and member variables from these classes to protected.
* Move some core methods from these classes to virtual, so the functionality can be
  extended.


## Rationale

This proposal will allow extending and experimenting with different server and
client implementations.


## Implementation

## Open issues (if applicable)
