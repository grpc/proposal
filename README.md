# gRPC RFCs
## Introduction
This repo contains the design proposals for substantial feature changes for
gRPC that need to be designed upfront. The goal of the upfront design process
is to:
- Provide increased visibility to the community on upcoming changes and the design considerations around them.
- Provide ability to reason about larger “sets” of changes that are too big to be covered either in an Issue or in a PR.
- Establish a consistent process for structured participation by the community on large changes, especially those that impact multiple runtimes and implementations.

## Prerequisites
This process needs to be followed for any significant change to gRPC that
needs design.
Changes that are considered significant can be:
- Features that need implementation across runtimes and languages.
- Process changes that affect how the gRPC product is implemented.
- Breaking changes to the public API (i.e. semver major changes).

## Process

1. Fork the repo and copy the template [GRFC-TEMPLATE.md](GRFC-TEMPLATE.md).
1. Rename it to ``$CategoryName-$Summary``, eg.: ``A6-client-retries.md`` (see
  category definitions below)
   - For language-specific proposals, include the name of the project:
     ``L##-$Language-$Summary``.  Canonical names: `c++`, `core`, `csharp`, `go`,
     `java`, `node`, `objc`, `php`, `python`, `ruby`.
1. Write up the RFC.
1. Submit a Pull Request.
1. Someone from gRPC team will be assigned as an APPROVER as part of this
review. Once the APPROVER is assigned, the OWNER needs to start a discussion on
[grpc-io](https://groups.google.com/forum/#!forum/grpc-io) and update the PR
with the discussion link. After this is done, the OWNER should update the gRFC
to the state of ``In Review``. It is expected that the APPROVER will help the
OWNER along this process as needed.
1. For at least a period of 10 business days (the minimum comment period),
it is expected that the OWNER will respond to the comments and make updates
to the RFC as new commits to the PR. Through the process, the discussion
needs to be kept to the designated thread in the mailing list in order to
avoid splintering conversations. The OWNER is encouraged to solicit as much
feedback on the proposal as possible during this period.
PR comments should belimited to formatting and vocabulary.
1. If there is consensus as deemed by the APPROVER during the comment period,
the APPROVER will mark the proposal as final and assign it a gRFC number.
Once this is assigned (as part of the closure of discussion), the OWNER will
update the state of the PR as final and submit the PR.
Commits must not be squashed; the commit history serves as a log of changes
made to the proposal.

## APPROVER
- By default ``a11r`` is the approver unless another approver is assigned
on a per-proposal basis.
- If the assigned APPROVER and the OWNER cannot satisfactorily settle an issue,
the final APPROVER is still ``a11r``.

## Proposal Categories
The proposals shall be numbered in increasing order.

- ``#An`` - Affects all languages.
- ``#Pnn`` - Affects processes, such as the proposal process itself.
- ``#Lnnn`` - Language specific changes to external APIs or platform support.
- ``#Gnnnn`` - Protocol level changes.

## Proposal Status
1. Every uncommitted proposal candidate starts off in the ``Draft`` state.
1. After it accepted for review and posted to the group, it enters the
``In Review`` state.
1. Once it is approved for submission by the arbiter, it goes into the
``Final`` state. Only minor changes are allowed (what qualifies as minor is
left to the APPROVER).
1. If a proposal needs to be revisited, it can be moved back to the ``Draft``
or ``In Review`` state. This can happen if issues are discovered during
implementation. At which point, the review process as described above must be
followed.
1. Once a proposal is ``Final`` and if it has been implemented by a language,
it can be updated to a status of ``Implemented`` with the list of languages
listed (listing versions is not required).
