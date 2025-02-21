A90: xDS ExtAuthz Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-02-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for the xDS ext_authz filter in the gRPC server.

## Background

The ext_authz filter provides support for servers making side-channel
call-outs to perform authorization decisions.

### Related Proposals: 
* A list of proposals this proposal builds on or supersedes.

## Proposal

Fields to support:
- grpc_service
- failure_mode_allow
- failure_mode_allow_header_add
- status_on_error
- allowed_headers -- note: may need to have some security control here?
- disallowed_headers
- encode_raw_headers (maybe only support true?)
==> CACHING!

Fields that we will not support:
- http_service -- don't plan to support this
- stat_prefix
- charge_cluster_response_stats
- emit_filter_state_stats

- filter_enabled -- seems like this is superceded by the "disable" field in the filter framework.  should we support that?
- filter_enabled_metadata
- deny_at_disable

- metadata_context_namespaces
- typed_metadata_context_namespaces
- route_metadata_context_namespaces
- route_typed_metadata_context_namespaces
- bootstrap_metadata_labels_key
- enable_dynamic_metadata_ingestion
- filter_metadata

Not sure:
- transport_api_version
- with_request_body -- would need to solve gRPC framing problem
- include_peer_certificate
- include_tls_session

Fields related to header mutation -- should we support these?
- clear_route_cache
- validate_mutations
- decoder_header_mutation_rules

### Temporary environment variable protection

[Name the environment variable(s) used to enable/disable the feature(s) this proposal introduces and their default(s).  Generally, features that are enabled by I/O should include this type of control until they have passed some testing criteria, which should also be detailed here.  This section may be omitted if there are none.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
