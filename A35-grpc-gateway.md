Integrate the grpc-gateway as a multi-language option to allow for REST endpoint support
----
* Author(s): Keith Adler (Talador12)
* Approver: a11r
* Status: Draft --- {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: Core (C++), Java, Python, and Go.
* Last updated: 2020-09-14
* Discussion at: https://groups.google.com/u/1/g/grpc-io/c/rEfqlLL9Emw (filled after thread exists)

## Abstract

Based on this [gRPC issue](https://github.com/grpc/grpc/issues/23979), the goal is to include the functionality of the [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) directly into the gRPC codebase across all languages. From @gnossen:

```
grpc-gateway is a separate binary/process from your server. It can be deployed alongside a server of any language, which (I assume) is what's happening in the gist you linked above. We have talked about in-library support for this sort of thing in the past. That way, you wouldn't have to orchestrate a separate gateway process. But that would be a pretty substantial effort.
```

As gRPC has grown in popularity and usage, an option to run in-library support would be a huge win for many teams. Instead of a sidecar, this gateway could be run alongside the gRPC server instance itself. As it would be a substantial effort, the background section will detail out the complexity of implementing across all languages and the steps required to start this process.


## Background

The PR on the grpc repo - [gRPC issue](https://github.com/grpc/grpc/issues/23979).

```
### Describe the solution you'd like
Add support for [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) as an option available in the [grpc python package](https://pypi.org/project/grpc/)

I have found a [python example](https://gist.github.com/doubleyou/41f0828e4b9b50a38f7db815feed0a6c) that claims to integrate this kind of gateway via python. Is this the accepted method?

### Describe alternatives you've considered
For non HTTP2 traffic, we are considering not using gRPC at all. We would prefer to use gRPC for both HTTP2 and HTTP1 via the grpc-gateway

### Additional context
Crossposted with the [grpc-gateway-repository](https://github.com/grpc-ecosystem/grpc-gateway/issues/1613)
```

Which led to:

```
grpc-gateway is a separate binary/process from your server. It can be deployed alongside a server of any language, which (I assume) is what's happening in the gist you linked above. We have talked about in-library support for this sort of thing in the past. That way, you wouldn't have to orchestrate a separate gateway process. But that would be a pretty substantial effort.
```

and:

```
The first step would be for the interested party to start a gRFC in https://github.com/grpc/proposal. There, you'd want to get relevant people from all of the languages to chime in on the requirements for the cross-language feature. Then, we'd need to get into per-language details. Finally, we'd need to ensure that support is added across the language matrix, which will entail at least three different implementations: Core (C++), Java, and Go.
```

### Related Proposals: 

* N/A - none was referenced by the gRPC team, although they have discussed implementing this feature informally in the past.

## Proposal

Add a [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) like component that is a separate binary/process from the gRPC server. It can be deployed alongside a server of any language. This should be in-library support, and not a `grpc-ecosystem` project. That way, you wouldn't have to orchestrate a separate gateway process. To implement this, it is necessary to ensure that support is added across the language matrix, which will entail four different implementations: Core (C++), Java, Python, and Go.

## Rationale

Using the `grpc-gateway` as a separate project is not always an option based on library usage. Projects that build on top of `grpc` often want the capability to offer HTTP1 traffic to clients, but do not have the ability to use in library solutions. Currently, most simply do not offer HTTP1 traffic. This excludes usage by several projects and companies.

The real trade off here is added functionality vs effort required. There will need to be a unified effort and lots of testing to ensure that the language matrix performs well on this use case. This could be a substantial effort.

We may be able to solicit help from the `grpc-gateway` contributors directly!

## Implementation

To implement this, it is necessary to ensure that support is added across the language matrix, which will entail four different implementations: Core (C++), Java, Python, and Go.

First draft of an order:

1. C++
2. Python
3. Go
4. Java

Looking for help on additional implementation requirements.

## Open issues (if applicable)

[gRPC issue](https://github.com/grpc/grpc/issues/23979)

[grpc-gateway-repository](https://github.com/grpc-ecosystem/grpc-gateway/issues/1613)