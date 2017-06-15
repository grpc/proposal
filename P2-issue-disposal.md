Github Issue Disposal Policy
-------------------------------------------------------
* Author(s): makdharma
* Approver: a11r
* Status: Draft
* Implemented in: N/A
* Last updated: 2017-06-15
* Discussion at: N/A

## Abstract

There are hundreds of open issues in the 3 gRPC repos, some open for over 2 years. gRPC team periodically spends valuable time scrubbing the issue list. This proposal lays down concrete criteria for disposing of issues filed by OSS community.

## Background

The github Issues database serves many purposes.

* Capture bugs
* Keep track of TODO
* Submit feature requests
* Ask questions (ad hoc documentation)

The team uses labels to informally classify issues. The default process is to keep the issue open unless we explicitly take care of all the work related with the issue. This process has an inherent tradeoff.

* Each open issue consumes some engineering bandwidth in periodic bug scrubs
* Leaving issue open adds uncertainty
* Closing an issue loses potentially important information
* The disposition process is not consistent across repos and users

Can we do better?

## Proposal

We propose the following.

### Keep status updated

Close following types of issues with explanation. Anyone can reopen if they disagree with the disposition.
* Issues that are fixed, or should have been fixed
* Issues that are waiting for more information and have not seen an update in over a month

### Correct classification

Issues that start out as bugs can turn into feature requests or document requests. Use following *mutually exclusive* labels to identify the nature of an issue - Question, Enhancement, Bug.

### Bring transparency to feature request handling

* Old feature requests (pending for over a year)
	* Make an honest assessment about feasibility and utility. Reject if appropriate with explanation. If the community feels strongly about the feature, anyone can open a new issue with added justification. This issue can then follow the New Issue path below.
	* If the feature request makes sense but we have no immediate plans, then consolidate in a separate document and close the issue. This is equivalent of adding "Someday" label.

Important note - Closing the issue does not mean that the request is invalid. It might be that it never rose in priority order, or that we don't have a strong signal from community about the importance of this feature.

* Newer feature requests
	* Outright reject where appropriate with explanation
	* Reconsider during quarterly planning for next 4 quarters
  * When a year passes without implementation, follow the Old feature request disposition
