Allow C++ in gRPC Core Library
----
* Author(s): ctiller nicolasnoble
* Approver: a11r
* Status: Draft
* Implemented in: n/a
* Last updated: April 1, 2017
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/yAg-ydC77aE

## Abstract

Allow C++ to be used in the gRPC Core library.

## Background

The gRPC Core library is currently implemented in C99, with a C89 public
interface. This gRFC proposes allowing C++ to be used within the core library,
but continues to restrict the public interface to C89.

### Related Proposals:

N/A

## Proposal

Allow C++11 usage in gRPC Core, with the following caveats:
- new/delete will be outlawed (in favor of wrappers that call into gpr_malloc,
  gpr_free - eg grpc_core::MakeUnique<>, grpc_core::Delete, grpc_core::New)
- exceptions and rtti will be disallowed
- standard library usage will be disallowed in favor of grpc core library
  substitutes
- all code will be required to live under a grpc_core namespace
- all resulting object code must be linkable with a normal C linker (no
  libstdc++ dependencies allowed)
- public API must continue to be C89

Provide a C++ utility library (much like GPR today) to assist implementation:
- grpc_core::UniquePtr<> (as a typedef for std::unique_ptr)
- grpc_core::Atomic<> (as a typedef for std::atomic)
- grpc_core::IntrusiveSharedPtr<>
- grpc_core::Vector<>
- grpc_core::IntrusiveList<>
- grpc_core::HashMap<>
- grpc_core::AVL<>
- grpc_core::Slice
- grpc_core::Closure
- grpc_core::ExecCtx
- grpc_core::Combiner
- grpc_core::Mutex

Where possible, typedef equivalent types in the C++ stdlib (this would only be
possible for header-only types).

## Rationale

Writing in C++ gives us several advantages:
- Safer memory management utilizing templated containers, smart pointers
- Simplify code by leveraging virtual functions and inheritance (the library
  contains many LOC that serve to emulate virtual functions and inheritance)
- Easier contribution (experience has shown it’s easier to attract C++ than C
  developers - and though we’ll be missing standard libraries, equivalent concepts
  should prove easier to find)
- C++ Performance: make it easier to drag lower level types into the C++ wrapper
  library, and reduce API friction
- Increase velocity by simplifying our idioms

## Implementation

1. Allow .cc files to be included in builds
   The “language” tag only becomes a linking hint for the build systems, and the C
   core maintains the status of a C library.
2. Convert lame_client.c to be C++ as a canary
3. Pause for one release cycle to validate assumptions that this is all safe
4. Start converting src/core/ext/client_channel/... to C++, as this library would
   gain the most from being implementable in C++ (especially lb, resolver
   interfaces)
5. On-demand during (4), implement needed C++ foundational libraries
6. Allow broader use of C++ within the library

## Open issues (if applicable)

- Build complexity: our wrapped languages will need to be able to handle compiling
  C++ in their build chains (though this is likely to need to happen for BoringSSL
  in the future also)
  - Most build systems just use the extension of the file to determine the
    compilation rule to apply.
- Build time will increase.
- Platform reach may decrease, but we feel this will not be significant.

