grpc-io-announce Mailing List
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: markdroth
* Status: Implemented
* Implemented in: N/A
* Last updated: 2024-09-30
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Create an "announce" mailing list to have a high-signal method to notify users
of important announcements, especially for security and reliability issues, so
they can take action.

## Background

When security and reliability issues are discovered, there hasn't been a good
way to quickly notify users they are impacted. grpc-io@googlegroups.com and
release notes are used, but the mailing list has too low of signal for many
users to watch and the release notes may take some time to be noticed.
Similarly, when important decisions are being made like dropping support for a
platform, it has been hard to get input or provide warning before the decision
is made and rolled out.

This proposal does not define how security, reliability, and other major
decisions are handled. It is limited to defining a mechanism of notifying users
about such important events.

## Proposal

A `grpc-io-announce` Google Group would be created to complement the current
[grpc-io group](https://groups.google.com/g/grpc-io). This would be a
low-traffic list of important announcements for when the gRPC project needs to
communicate with its users, with much higher signal/noise ratio than the grpc-io
mailing list used today.

gRPC has high-importance communications other than just, for example, security,
and this group would serve those purposes as well. But the signal/noise ratio
must be kept very high to remain useful to gRPC users. The list will be
moderated to prevent inappropriate or accidental emails. The initial moderators
will be the gRPC TLs and management, but can morph over time as helpful.

To keep signal/noise ratio high, the list can be used to announce:

1. Medium or serious reliability issues with workaround or fix available
2. Medium or serious security issues with workaround or fix available
3. Plans for major version releases in any supported language
4. Plans for dropping platform support in any supported language
5. Major project and governance announcements, like when gRPC’s license changed
to Apache 2 or when gRPC became part of the CNCF
6. At most once-a-year user survey


It would not be used for:

1. Conversations about any of the topics. That should happen on grpc-io, GitHub
issue, or similar. Emails from the group will have `reply-to:` set to
grpc-io@googlegroups.com
2. Regular releases
3. New features

There is no requirement to use the list. Maintainers are encouraged to not use
the list when a notification would add too much noise for its benefit. This
would generally be the case for regularly-scheduled events.


### Process

**(If dealing with a security vulnerability you can use this process in part,
but security vulnerability processes take priority. Make sure appropriate
embargos are expired, including any necessary rollouts to PROD, before making
information public.)**

Overview:

1. Compose an email to `grpc-io-announce@googlegroups.com`
2. Have the email reviewed by another person
3. Send the email to grpc-io, BCCing `grpc-io-announce@googlegroups.com`
4. Update Github release notes of impacted versions
5. Follow-up notifications as necessary

When composing an email to `grpc-io-announce@googlegroups.com`, you should:

*   Clearly define how the user should determine if their application is at risk
    *   e.g., “grpc-java users of grpc-xds 1.43.0 and later are impacted. You can run `maven dependency:tree` or `gradle dependencies --configuration=runtimeClasspath` to find the versions used by a particular project”
    *   e.g., “grpc-node users running on Node.js 14”
*   Define the severity of the problem, and how to know if the issue is occurring
    *   e.g., “This bug may cause all RPCs on a channel to fail with ‘UNAVAILABLE: Handshake failure’”
*   Make the email actionable. For fixes, clearly define how the user should resolve the problem
    *   e.g., “Upgrade grpc-xds to 1.47.1 or later. If grpc-xds is a transitive dependency, adding a direct dependency on grpc-xds will upgrade the version. Maven users are strongly encouraged to use Maven Enforcer’s requireUpperBoundDeps to avoid accidental version downgrades”
    *   e.g., “Upgrade to Node.js 18 or later, preferably Node.js 20 or later”
*   Define a discussion forum for questions, concerns, and replies. This will generally be an existing GitHub issue or new grpc-io thread

Emails sent to `grpc-io-announce@googlegroups.com` should be addressed as:

```
To: grpc-io@googlegroups.com
BCC: grpc-io-announce@googlegroups.com
Subject: gRPC $LANG Lorem Ipsum
```

Such addressing allows replies to be seen on grpc-io, prevents replies from spamming the `grpc-io-announce@` moderation queue, and allows users to quickly ignore announcements for irrelevant languages. Use a single email, independent of the number of implementations impacted.

As that is sent, update the Github release notes of impacted releases to include
a warning. If too many releases are impacted, just update newer releases.

Severe issues may require an announcement early, with only a workaround or a few releases available. Follow-up emails on `grpc-io-announce@` may occur to provide updated information, but they should be few and batched. Finer-grained notification should be on the Github issue or grpc-io@.


### Example

[Example notification email and follow-up](https://groups.google.com/g/grpc-io/c/roNyXX5pEAU/).

## Rationale

The main alternative is "do nothing" and use what we already have. An announce
list was originally proposed when gRPC became public (before 1.0), but there
wasn't wider concensus it would be useful. Since gRPC's user base is now quite
large, recent experience has demonstrated it will very likely be useful now.

The design heavily optimizes making the list useful to those subscribed, so they
choose to subscribe and remain subscribed. It will likely serve the greatest
benefit when widely subscribed.

## Implementation

A `grpc-io-announce@` mailing list has already been created, to avoid it being
registered by a non-maintainer. @ejona86 will configure it and add the initial
moderators.

An email will be sent on grpc-io mailing list to encourage people to subscribe
and encourage those they know that aren't on grpc-io to subscribe. gRPC
implementations will be encouraged to add a note in their next minor version's
release notes notifying users of the mailing list. The implementations will also
be encouraged to add a link in their README.

