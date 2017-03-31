Backward Compatibility Management
----
* Author(s): Muxi Yan, Feng Li
* Approver: a11r
* Status: Draft
* Implemented in: All languages supporting gRPC server
* Last updated: 03/30/2017
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

1. Provides a way to identify the gRPC client's versions
2. Provides a way to manage workarounds on gRPC servers so that they are compatible with previous versions of gRPC clients

## Background

As more versions of gRPC are released to users, different users may use different versions of gRPC at the same time. However these versions of gRPC may not have the same set of features available and may contains bugs to make them behaves incorrectly.

gRPC should be able to do the following:
* When a gRPC client is of old version and does not support a feature, the server can identify the client's lack of such capability and disable corresponding features appropriately, if needed.

Below use cases have happened multiple times and is expected to continue happening in the future. Known cases are:
* Compression was not supported in gRPC Objective C with Cronet due to a bug and was fixed in version v1.3.0-dev. The gRPC endpoint on the other side, ESF server in this case, is unable to enable compression only for clients of v1.4.0-dev or higher due to lack of this mechanism being proposed.
* An old gRPC Objective C client (without Cronet) will crash when getting compressed response.
* ctiller@ mentioned that there are also some flow control behavior get corrupted due to buggy clients.
* I remember there are also some compatibililty issue across multiple languages, like the `GO_AWAY` frame issue with golang. We may consult different language teams to add more in this list.

### Related Proposals: 
N/A

## Proposal
The major considerations of the design are:
* Backward compatibility issues should be managed by gRPC server instead of user
* Servers that do not care about any backward compatibility issue would not need to do anything special
* Servers that care about a particular backward compatibility issue can invoke an API of gRPC library to configure the server so that the server will apply workaround to that issue based on client's version.

### Client version identification
gRPC server should identify the version of a client by analysing the `user-agent` header field on the server side. Current format of `user-agent` is well defined by the doc [PROTOCOL-HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md). With this definition we can extract gRPC language and version from client's `user-agent` string and identify user's compatibility issues.

### Server backward compatibility management
gRPC server will maintain a list of server workarounds for backward compatibility issues. These workarounds are disabled by default for performance consideration. If a user cares about a backward compatibility issue, they should be able to enable the workaround. In this case gRPC server will identify the version of each client and determine whether the workaround should be applied.

#### Workaround list
The workaround list on gRPC server is a map from an integer `workaround id` to a `version comparing function`:
| Workaround id | Version comparing function |
| ------------- | -------------------------- |
| 1             | `bool higher_than_v_1_2(uint32_t id, grpc_slice user_agent)` |
| 2             | `bool not_objc_or_higher_than_v_1_3(uint32_t id, grpc_slice user_agent)` |
| ...           | ... |

* `Version comparing function` should be a function which takes a client `user-agent` string as parameter and returns a bool value indicating whether a workaround should be applied
* The workaround list should be well documented in gRPC repository. The doc should include the items of the list, as well as detailed description of each workaround and its corresponding compatibility issue, so that users can identify those workarounds they want to enable
* The list should be compiled into gRPC server and shipped to users in releases
* An item is add to the list when a backward compatibility issue is raised and a workaround is implemented on the server
* Updated list will be pushed to users in the next release (or next import for Google3)
* An item should be removed from the list after 6 months (roughly 4 minor version releases)

The list should be implemented and maintained by the three core languages for interoperability. Wrapping languages should use surface APIs (below) to access the table.

#### APIs
```
GRPCAPI void enable_workaround(uint32_t id);
```
* This API allows user to enable a particular server workaround for incompatible clients.
* It should be exported by core languages and wrapping languages which allow users to create gRPC server. 

```
GRPCAPI bool is_client_compatible(uint32_t id, grpc_slice user_agent);
```
* This API allows gRPC server to check whether a workaround should be applied for a particular client, by checking the client's `user-agent`.
* Returns `true` if
  1. The workaround `id` is not enabled by `enable_workaround`, or
  2. `Version comparing function` of workaround `id` returns `true` for the particular `user-agent`.
* It should be exported to wrapping languages as a surface API.
* For performance, this function should be called as few times as possible, e.g. once for each client and each workaround at channel creation.

## Rationale
### User-agent field
Since gRPC currently follows a 6-weeks release cycle, using current `user-agent` field to identify a client's version should allow users to be blocked by a backward compatibility issue for at most 6 weeks. This should be acceptable in most cases.

### Version comparing function
Instead of determining client's compatibility by a minimum version number, we decided to make it more general with `version comparing function`. This allows more flexibility in determining client's compatibility, such as including client's language into consideration.

### Performance
Current API requires a client's `user-agent` string to be checked against the version comparing functions of all items in the workaround list, involving a lot of (likely duplicated) string processing. Depends on how much performance we need, an alternative is to extract key information from `user-agent` (e.g. gRPC version, language, transport, etc.) and use them as parameter of `version comparing functions`. It is a tradeoff of flexibility to performance that we may want to consider.

## Implementation
1. Implement workaround list and surface APIs in C (mxyan@), Java (TBA) and Go (TBA). 
2. Implement `enable\_workaround` API for user in wrapping languages providing server (TBA).
3. Create workaround list doc on gRPC Github repo (mxyan@).

## Open issues (if applicable)
Currently, gRPC allows user to add custom strings into `user-agent` field. As far as I know all the languages prefix the user's custom `user-agent` in front of gRPC`s `user-agent` string. Some languages, however, allows user to completely overwrite the `user-agent` string, which are not supposed to be. These methods are not documented, but we do not know if users have taken advantage of it. This is a risk associated with identifying client's version with `user-agent` field.
