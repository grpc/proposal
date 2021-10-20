A47: xDS Federation
----
* Author(s): Mark D. Roth (@markdroth)
* Approver: ejona86, dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-10-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

[xRFC TP1][xRFC TP1] describes a new naming scheme for xDS resources
where resource names are encoded as URIs with an `xdstp` scheme.  This
new naming scheme is the cornerstone to support **federation**, because
it allows xDS clients to get different resources from different xDS
servers without concern about naming conflicts.

This document describes how gRPC will support xDS federation and the new
resource naming scheme.

## Background

[An introduction of the necessary background and the problem being solved by the proposed change.]


### Related Proposals: 
* [xRFC TP1: xdstp:// structured resource naming, caching and federation support][xRFC TP1]

[xRFC TP1]: https://github.com/cncf/xds/blob/main/proposals/TP1-xds-transport-next.md

## Proposal

[A precise statement of the proposed change.]

### Temporary environment variable protection

[Name the environment variable(s) used to enable/disable the feature(s) this proposal introduces and their default(s).  Generally, features that are enabled by I/O should include this type of control until they have passed some testing criteria, which should also be detailed here.  This section may be omitted if there are none.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
