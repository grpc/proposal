Java: Drop support for Protobuf's Javanano
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: a11r
* Status: In Review
* Implemented in: Java
* Last updated: 2019-04-22
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Remove support for the
[Javanano](https://search.maven.org/search?q=g:com.google.protobuf.nano)
Protobuf API.

## Background

Javanano ("nano" or "nano proto") was an Android-centric API that uses open
structures to optimize dex method count and size. It was released in protobuf
3.0.0 alpha, and apparently a 3.1.0 (although this may have been a mistake, as
it wasn't re-released later and it wasn't in the release notes). This nano API
is very cumbersome and ugly. Javanano can't really be used by libraries, as
there are a _lot_ of configuration options for the code generator and they
often need to be tuned for the particular app.

[Protobuf Lite](https://search.maven.org/search?q=g:com.google.protobuf%20a:protobuf-lite)
is a subset of the full protobuf API also intended for Android. After tools
like ProGuard, it can have a similar size to Javanano. There is also work that
causes it to be better optimized for Android than nano.

Upstream protobuf removed support for nano in 3.6, and [is encouraging users to
use lite instead of
nano](https://github.com/protocolbuffers/protobuf/issues/5288). Since nano was
included in protoc and not as a separate plugin, this has been preventing
grpc-java [from upgrading protoc in certain
circumstances](https://github.com/grpc/grpc-java/pull/5320). This problem will
get worse with time.

### Related Proposals: 

None.

## Proposal

* grpc-java completely drops support for protobuf nano
  * grpc-protobuf-nano is deleted from the source and no longer released
  * protoc-gen-grpc-java drops support for the 'nano' flag
  * grpc-bom will remove its reference to grpc-protobuf-nano

## Rationale

grpc-protobuf-nano depends only stable APIs. While it depends on
MethodDescriptor.Marshaller, [Marshaller is to be stabilized in
1.21](https://github.com/grpc/grpc-java/pull/5617) with the same API it has had
since 1.0. That means that even after it is no longer updated, the old releases
will continue working.

protoc-gen-grpc-java's generated code has always been forward-compatible; it
does not rely on unstable APIs. So even after newer versions drop nano support,
older releases will continue working.

This means a simple deletion of the code in grpc-java allows grpc-java to move
forward with newer versions of protobuf/protoc while letting existing users
continue as they were, although they should strongly consider migrating to lite
(although should also be aware that the [lite API is considered unstable
API](https://github.com/protocolbuffers/protobuf/blob/v3.7.1/java/lite.md)).

Given how clean this proposal is, no alternatives were seriously considered.

## Implementation

The PR to implement this is already available as grpc/grpc-java#5622.

## Open issues (if applicable)

None.
