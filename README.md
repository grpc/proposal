# gRPC RFCs

## Introduction

Please read the gRPC organization's [governance
rules](https://github.com/grpc/grpc-community/blob/master/governance.md) and
[contribution
guidelines](https://github.com/grpc/grpc-community/blob/master/CONTRIBUTING.md)
before proceeding.

This repo contains the design proposals for substantial feature changes for gRPC
that need to be designed upfront. The goal of the upfront design process is to:

- Provide increased visibility to the community on upcoming changes and the
  design considerations around them.
- Provide ability to reason about larger “sets” of changes that are too big to
  be covered either in an Issue or in a PR.
- Establish a consistent process for structured participation by the community
  on large changes, especially those that impact multiple runtimes and
  implementations.

## Prerequisites

This process needs to be followed for any significant change to gRPC that needs
design. Changes that are considered significant can be:

- Features that need implementation across runtimes and languages.
- Process changes that affect how the gRPC product is implemented.
- Breaking changes to the public API (i.e. semver major changes).

## Process

1. Fork the repo and copy the template [GRFC-TEMPLATE.md](GRFC-TEMPLATE.md).
1. Write the RFC.
1. Rename it to ``$CategoryName##-$Summary``, eg.: ``A6-client-retries.md``
   (see category definitions below)
   - For language-specific proposals, include the name of the language:
     ``L##-$Language-$Summary``. Canonical names: `core`, `cpp`, `csharp`, `go`,
     `java`, `node`, `objc`, `php`, `python`, `ruby`.
   - To determine the number to use for your proposal, view all PRs (open and
     closed), sorted by creation date ([link](
     https://github.com/grpc/proposal/pulls?q=is%3Apr+sort%3Acreated-desc)).
     Find the first _new_ proposal PR of the same type, and use the following
     number.
1. Submit a Pull Request.
1. Someone from the gRPC team will be assigned as an APPROVER as part of this
review.
1. For a period of at least 10 business days (the minimum comment period), it is
expected that the OWNER will respond to the comments and make updates to the RFC
as new commits to the PR. The OWNER is encouraged to solicit as much feedback on
the proposal as possible during this period.
1. If there is consensus as deemed by the APPROVER during the comment period,
the APPROVER will approve the PR on GitHub. The PR will then be merged by either
the OWNER, if possible, or the APPROVER otherwise.

## APPROVER
- By default ``a11r`` is the approver unless another approver is assigned
on a per-proposal basis.
- If the assigned APPROVER and the OWNER cannot satisfactorily settle an issue,
the final APPROVER is still ``a11r``.

## Proposal Categories
The proposals shall be numbered in increasing order.

- ``An`` - Affects all languages.
- ``Pnn`` - Affects processes, such as the proposal process itself.
- ``Lnnn`` - Language specific changes to external APIs or platform support.
- ``Gnnnn`` - Protocol level changes.

## Updating proposals
When updating a proposal, the PR description should be named as follows:

    $CategoryName## update: <description of change>``

If the update is minor (what qualifies as minor is left to the APPROVER), the PR
may be approved and merged at the OWNER's and APPROVER's convenience.
Otherwise, the original process's 10 day comment period must be observed.
