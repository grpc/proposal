P5: JDK Version Support Policy
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: dapengzhang0
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: Java
* Last updated: 2021-12-23
* Discussion at: https://groups.google.com/g/grpc-io/c/S1biTro_sbE

## Abstract

Define the guidelines for dropping older JDK versions to make the process easier
and more predictable. Android API level support is separate and is not impacted.

## Background

Java version upgrades have historically been disruptive. Java versions were
released years apart and while they are mostly backward compatible, behavior
changes impact many users. There has also been a long history of frameworks that
integrate deeply with Java and libraries abusing implementation details (e.g.,
Unsafe, `sun.*` classes). But you also have more run-of-the-mill issues like
`enum` becoming a keyword or updates for new bytecode versions. After all that
you may have also seen a change in performance because an application was
optimized to the previous version.

Often many of these issues were easy to resolve by upgrading a transitive
dependency. But when a transitive dependency was no longer supported (e.g.,
because it has been replaced with a new major API version) the upgrade to a new
Java version becomes a major undertaking. For library authors it was hard to
know when it was "safe" to drop support for an old JDK version, and may find out
the issues caused years after the decision. This created an ecosystem of sincere
fear, uncertainty, and doubt where library owners did not want their
dependencies to drop older JDK support, even though the benefit was unclear. It
was rightly known that _some_ user was certainly still using an older JDK
version, but it was unclear how many and whether they ever would need a new
version of a library.

That process has been changing since Java 8, especially with the new more rapid
releases and planned LTS releases and reduced support timelines. Java releases
have been offering features that have been encouraging upgrades and Java is
hiding more of its internals to make upgrades easier. Running separate
applications in a single, large application server is less common which allows
upgrading pieces individually. The ecosystem has gotten more practiced at
supporting new versions, less willing to be held back, and better at
compartmentalizing legacy applications.

gRPC is widely used and is lower in the dependency tree, so it has more
conservative support. gRPC Java 1.0 supported Java 6, which was dropped
following discussion in grpc/grpc-java#3961 because Java 6 was becoming
unsupportable as the compiler was unmaintained and other libraries were dropping
support. The discussion for dropping Java 7 is happening at grpc/grpc-java#4671
and is necessary for reasons similar to Java 6. However, both processes were
time-consuming and had no predictable timeline.

gRPC has suffered from supporting old Java versions. There are hidden
maintenance costs, but it is easy for someone to argue those should just be paid
(either because of gRPC's scale or because they are not paying those costs).
However, the public API surface has been limited which directly impacts users.
The most obvious example of this is the lack of generated `interface`s for
servers. Adding methods to an existing service must be backward-compatible, and
the only way to do that before Java 8 is with an abstract class. But once we can
use Java 8 language features we can use `default` methods in an `interface`.

Android support is tracked separately and generally follows Google Play
Services's supported API level, for security and as a reasonable guideline of
what is worth supporting. Android support is not impacted by this gRFC as it has
different options available to it (e.g., desugaring) and does not align cleanly
to JDK versions.

## Proposal

gRPC Java may drop support for a JDK version when that version is no longer
receiving [Premier Support from Oracle][Oracle Java support]. When doing so,
gRPC maintainers will publicize a recent minor version release (but not
necessarily the most recent) to be the suggested release for users on the older
JDK. That release is intended to be a gathering point for the gRPC ecosystem
users that need the older JDK support.

gRPC maintainers will allow changes to the release branch for compatibility,
bug, and security fixes and will aid the community in code reviews and
privileged operations necessary for a release. But gRPC maintainers would not be
expected to maintain it longer than the [normal support policy][gRPC support
policy]; the community as a whole would be responsible for maintenance.

There is no requirement to increase the major version for the release dropping
support for an older JDK.

[Oracle Java support]: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
[gRPC support policy]: https://grpc.io/docs/what-is-grpc/faq/#how-long-are-grpc-releases-supported-for

## Rationale

Following Oracle's Premier Support means Java 8 may be dropped in 2022 and
Java 11 may be dropped in 2023. That is rapid compared to historical norms. But
compared to other languages, excepting C/C++, it is still conservative as JDK
releases are supported for five years and two years after its replacement is
available.

There are providers of JDK support other than Oracle and Oracle offers extended
support. However, Oracle's support timelines seem fair and are frequently
referenced in the Java ecosystem. So even when not using Oracle's JDK, the
support dates are a good guideline.

Java has been providing strong incentives to upgrade in the form of new features
and performance. Developers want those benefits and so push their organizations
to upgrade. It is in everyone's best interest to upgrade regularly, as otherwise
Java comes dangerously close to ossifying as we have seen with the slow
historical upgrades.

Applications running on JDK versions not receiving Premier Support are much less
likely to be actively developed and so are less likely to need new features.
Some orphaned libraries have caused issues in the ecosystem because they had
_no_ updates, no matter the reason. Allowing further patch releases seems enough
to service these users, and having those users help with maintence puts them in
a better position to balance the cost of upgrading to a newer JDK. It also
provides a signal that users are still using the older JDK version.

Since the API does not change when dropping support for a JDK, [semver][] does
not require changing the major version. We similarly do not change the major
version when dropping support for older Android versions. gRPC is dropping
support for the old JDK in part because few people are expected to be using it
(gRPC is on the "following" edge, not "leading" edge); if a substantial
percentage of users are impacted then the approach may need to change or see
refinement.

[semver]: https://semver.org/spec/v2.0.0.html
