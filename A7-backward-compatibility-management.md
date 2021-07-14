Backward Compatibility Management
----
* Author(s): Muxi Yan, Feng Li
* Approver: ctiller
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

### Server backward compatibility workarounds
When a backward compatibility issue emerges, gRPC servers may implement workaround for these issues. Each workaround should be documented in a gRPC doc and assigned a unique ID in the format `WORKAROUND_ID_*` where the last part of the ID is a brief description of the workaround. In all gRPC server languages, the workarounds IDs should be listed in an enumeration named `workaround_ids` (may be tweaked following naming convention of each language) exported to users:
```
enumeration workaround_ids {
  WORKAROUND_ID_CRONET_COMPRESSION,
  WORKAROUND_ID_VERSION_MISMATCH,
  ...
}
```

Workarounds should be implemented in a way that they can be enabled or disabled. A workaround should be executed only when it is in enabled state. In considerantion of performance, all workarounds are disabled by default. gRPC servers should provide users an API to enable a workaround by its ID at construction time:
```
enable_workaround(id : workaround_ids) : void
```
The parameter `id` specifies the ID of the workaround to be enabled.

### Workarounds management
All backward compatibility workarounds should be documented in the file `grpc/docs/workaround_list`. The doc contains a list of workarounds implemented in gRPC servers. Each item in the list includes the ID of the workaround, corresponding issue it resolves, how the workaround is being implemented, the date when it is included, current status of the workaround (e.g. implemented in which gRPC servers), and other remarks. Users may refer to this doc to find and then enable a workaround they care about. 

When a backward compatibility issue emerges, the gRPC team member who is assigned to this issue should add an item to the workaround list doc, then implement corresponding workaround and/or assign the implementation in various languages of gRPC servers to one or more other team members.

When a workaround is implemented in one language of gRPC server, the implementer should update current status of the workaround item in the doc.

An item in the workaround list, together with the corresponding workaround in code base, should be removed 6 months (roughly 4 minor version releases) after it was added.

## Rationale
### Enable/Disable workarounds
The design of enabling/disabling workarounds are based on performance considerations. We do not want a user who does not care about a particular workaround to suffer from performance cost on turning on that workaround, so all workarounds are disabled by default. Only when a user needs a workaround that they enable it with the API provided by server.

## Implementation
1. Implement in C (mxyan@), Java (TBA) and Go (TBA). 
2. Implement in wrapping languages providing server (TBA).
3. Create workaround list doc on gRPC Github repo (mxyan@).

Fireball is currently waiting for this implementation in their ESF endpoint, so the need of implementing this in C and C++ takes precedence over other languages.
