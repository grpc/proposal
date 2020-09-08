Remove grpc_channel_ping from Core Surface API
----
* Author(s): yashykt
* Approver: markdroth
* Status: Final
* Implemented in: Core https://github.com/grpc/grpc/pull/23894
* Last updated: 2020-08-19
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/suC7gw_WOa4

## Abstract

Remove `grpc_channel_ping` from the core surface API.

## Background

`grpc_channel_ping` allows for the application to send a ping over the channel.

### Related Proposals:

N/A

## Proposal

Remove `grpc_channel_ping` from the core surface API.

## Rationale

`grpc_channel_ping` is not used outside of tests, so there is no reason for it to be a part of the surface API.

## Implementation

Core: https://github.com/grpc/grpc/pull/23894

## Open issues (if applicable)

N/A

