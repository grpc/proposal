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
When a backward compatibility issue emerges, server may implement workaround for these issues. When building a server, the workarounds in the server should be listed in an enumeration, each of them identified by an ID. The server should support turning on/off each of these workarounds by ID at run time. All workarounds should be disabled by default. 

#### API
A server should export a single API to users, which allows them to enable/disable a workaround by its ID.

#### Workaround management
Each workaround should be well documented in gRPC repository. The doc should include a list of workarounds. Each item includes the ID of the workaround, corresponding issue it resolves, how the workaround is being implemented, and the date when it is included. Users may refer to this doc to enable a workaround they care about. 

An item is added to the list of workarounds when it is implemented in the server. An item in the list of workarounds should be removed 6 months (roughly 4 minor version releases) after it was added.

## Rationale
### Enable/Disable workarounds
The design of enabling/disabling workarounds are based on performance considerations. We do not want a user who does not care about a particular workaround to suffer from performance cost on turning on that workaround, so all workarounds are disabled by default. Only when a user needs a workaround that they enable it with the API provided by server.

## Implementation
1. Implement in C (mxyan@), Java (TBA) and Go (TBA). 
2. Implement in wrapping languages providing server (TBA).
3. Create workaround list doc on gRPC Github repo (mxyan@).
