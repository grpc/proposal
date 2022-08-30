Remove custom allocation function overrides from core surface API
----
* Author(s): veblush
* Approver: mark
* Status: Approved
* Implemented in: https://github.com/grpc/grpc/pull/20462
* Last updated: 2019-10-09
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/kDswzg-cEZU

## Abstract

Remove `gpr_get_allocation_functions` and `gpr_set_allocation_functions`
from the core surface API.

## Background

Since gRPC Core was allowed to use the C++ standard library, it's getting
harder to keep using the gRPC allocator which can be overriden by users.
Previously, it was fairly straightforward to use a family of gpr memory
allocation functions such as `gpr_malloc` and `gpr_free` because most of
the code was written in C language.
Once C++ started being used heavily, however, using it makes code harder to
read and sometimes impossible to do it.
Instead of maintaining these functions, this proposes to remove them.

### Related Proposals:

N/A

## Proposal

Following functions managing custom memory allocator will be removed.

- `gpr_get_allocation_functions`
- `gpr_set_allocation_functions`

gRPC memory allocation functions such as `gpr_malloc` and `gpr_free` will
remain with this change because they have been used when data allocated in
user applications is passed into gRPC Core such as `metadata`.

## Rationale

C++ provides a way to use a custom memory allocator but it usually requires to
enter more code and makes it harder to read. 

This is an example of how this looks like with an allocator.

```
// New<T> allocates memory with custom allocator and calls constructor of T.
auto a = New<SomeClass>(obj);

// Delete<T> calls destructor of T and frees the memory associated with it.
Delete<SomeClass>(a)

// All container classes should be instantiated with a custom allocator.
std::map<int, std::string, Allocator<std::pair<const int, std::string>>> m;

// unique_ptr should carry special Delete function to use an allocator.
grpc_core::UniquePtr<char> a = grpc_core::MakeUnique<char>("a");
```

This can be simplified by not supporting a custom allocator. Note that the 
standard way of using`delete` doesn't mandate to specify the type of instance
 unlike `gprc_core::Delete`.

```
// plain new
auto a = new SomeClass(obj);

// plain delete
delete a;

// plain map
std::map<int, std::string>

// plain unique_ptr
std::unique_ptr<char> p = std::make_unique<char>("a");
```

In addition to this, some of C++ libraries cannot use custom allocators.
For example, `std::function` doesn't support an allocator. Moreover,
none of gRPC wrapped libraries including gRPC C++ doesn't support this.
As a result, memory allocation can be done either in a built-in allocator
or a custom allocator. This can be misleading to developers.

## Implementation

Core: https://github.com/grpc/grpc/pull/20462

## Open issues (if applicable)

N/A
