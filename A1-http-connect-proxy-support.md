## HTTP CONNECT Proxy Support

*   Author(s): Mark D. Roth (roth@google.com), Eric Anderson (ejona@google.com)
*   Approver: a11r
*   Status: Implemented
*   Implemented in: C-core (Java and Go are in progress)
*   Last updated: 2017-03-20
*   Discussion at:
    https://groups.google.com/d/topic/grpc-io/zG7iYpDw6lk/discussion

## Abstract

Describes what types of TCP-level proxy setups gRPC will support and how those
setups interact with client-side per-call load balancing policies (e.g.,
`grpclb` or `round_robin`).

## Background

It may be useful to read the following docs before this one:

-   [gRPC load balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)
-   [gRPC name resolution](https://github.com/grpc/grpc/blob/master/doc/naming.md)
-   [gRPC service config](https://github.com/grpc/grpc/blob/master/doc/service_config.md)

A TCP-level proxy is one that does not know anything about gRPC. It accepts a
TCP connection from a client, creates a new TCP connection to another server,
and forwards all bytes from the client to the server, without actually
understanding anything about the contents of those bytes.

Note that client-side per-call load-balancing within gRPC requires the client to
establish connections to individual backends. Because of this, use of a
TCP-level proxy may limit the ability to do client-side per-call load balacing
in some environments.

### Related Proposals

-   [gRFC A2: Service Configs in DNS](https://github.com/grpc/proposal/pull/5)
-   [gRFC A5: Encoding grpclb data in DNS](https://github.com/grpc/proposal/pull/10)

## Proposal

gRPC will support TCP-level proxies via the HTTP CONNECT request, defined in
[RFC-2817](https://tools.ietf.org/html/rfc2817).

### Use Cases

We are aware of the following use-cases for TCP-level proxying with gRPC:

1.  A corp environment where all outbound traffic must go through a proxy. This
    environment has the following properties:

    -   Name resolution is done by the proxy, either because external DNS is not
        resolvable internally, or because the proxy requires the CONNECT request
        to use a hostname instead of an IP address.
    -   Internal traffic does not need to use the proxy, but external traffic
        does.
    -   In some environments, there will not be any gRPC traffic to any internal
        servers, in which case we can unconditionally send *all* connections
        through the proxy. This configuration is generally triggered by some
        centralized signal, such as the `http_proxy` environment variable or
        Java system properties.

2.  A partially protected environment, where access to certain addresses must go
    through a proxy. This environment has the following properties:

    -   Name resolution of protected servers works normally, and the proxy
        allows the CONNECT request to use an IP address instead of a hostname.
    -   Only requests for certain hosts must go through the proxy. Requests to
        other servers work without the proxy.
    -   Custom logic is used to determine which hosts the proxy will be used
        for.

3.  An environment where a service is accessible from the Internet but its
    internal architecture should not be exposed publicly. This environment has
    the following properties:

    -   Name resolution of internal names does not work externally (i.e., all
        internal resolution must be done via the proxy).
    -   Only requests for services behind the proxy will go through it.

### Proposed Functionality

We propose the following functionality to support these use cases:

-   We will have some internal state (e.g., a channel arg in C-core) that both
    triggers the use of the HTTP CONNECT handshaker and specifies the argument
    to be used in the CONNECT request.

-   We will support a new type of hook called a *proxy mapper*, which offers two
    points at which proxies can be selected:

    -   Right before the server name is resolved, the proxy mapper can be used
        to programmatically override the name that will be resolved. It will
        also be able to set the internal state to trigger use of the HTTP
        CONNECT handshaker.
    -   Right before we connect to the target address, the proxy mapper can be
        used to programmatically override the address that we will connect to.
        It will also be able to set the internal state to trigger use of the
        HTTP CONNECT handshaker.

### Addressing Use Cases

This new functionality will be used to address the three use-cases described
above, as follows.

#### Case 1

Case 1 will be addressed using the proxy mapper hook for overriding the server
name to resolve. Sites that may want to send only external servers through the
proxy can implement their own proxy mapper to do this. We will provide a default
proxy mapper implementation that looks for the existence of the `http_proxy`
environment variable (or equavalent signal). If that signal is present, it will
do the following: - Ask the resolver to resolve the proxy name instead of the
server name requested via the client API. (Note that this will cause us to
create a single subchannel to the proxy, as opposed to one subchannel for each
backend server.) - Add the state described above triggering the use of the HTTP
CONNECT handshaker, with the argument set to the server name requested via the
client API.

Note that in this case, because the client cannot know the set of server
addresses, it is impossible to use the normal gRPC client-side per-call load
balancing. It *is* possible to do load balancing on a per-connection basis, but
that may not spread out the load evenly if different clients impose a different
amount of load. (Note that even if we established multiple connections to the
proxy, we would have no guarantee that each connection would go to a different
backend server.)

Also, note that in this case, because the client is not resolving the server
name itself, it will not have access to the service config.

#### Case 2

This case will be addressed using the proxy mapper hook for overriding the
address to connect to. When the proxy mapper sees an address that should go
through the proxy, it will do the following: - Set the target address to the
proxy address. - Add the additional state described above to trigger the use of
the HTTP CONNECT handshaker, with the argument set to the original target
address. Note that in this case, the server name used in the HTTP CONNECT
request will be an IP address and port, not a hostname (i.e., we are not relying
on the proxy to do name resolution for us).

In contrast to case 1, in case 2, it *is* possible to do the normal gRPC
client-side per-call load balancing, because the client does know the set of
server addresses. The fact that the connections to those individual addresses go
through a TCP-level proxy does not interfere with the client opening a separate
subchannel to each address.

In this case, because the client is resolving the server name itself, the client
will have access to the service config as usual.

#### Case 3

This case is similar to case 1 in that the resolver cannot return the actual
addresses of the servers. However, we (ab)use the gRPC load-balancing design to
work around this, so that it looks more like case 2.

In this case, the resolver must return the address of the proxy, but with the
`is_balancer` bit set. This can be done either by writing a custom resolver or
by publishing a DNS SRV record for the server name's `grpclb` service that
points to the proxy name (as per
[gRFC A5: Encoding grpclb data in DNS](https://github.com/grpc/proposal/pull/10)).

Next, we use the proxy mapper hook for overriding the address to connect to. The
proxy mapper implementation will have to detect two types of addresses: - When
it sees the proxy address, it will set the HTTP CONNECT argument to the original
server name (which will cause the proxy to establish a connection to one of the
load balancers). - When it sees any internal address (i.e., the addresses
returned by the load balancer), it will replace it with the proxy address and
set the HTTP CONNECT argument to the internal address.

Note that this will work only if the `grpclb` load balancing policy is in use;
it will not work with client-side policies like `round_robin`.

Also, note that if a custom resolver is implemented, then the client may not
have access to the service config. However, if the DNS SRV record is published,
then the service config will be returned normally.

## Rationale

The proposed design allows two different ways of triggering use of the HTTP
CONNECT handshaker code, which addresses the needs of the use-cases described
above.

There is one down-side to the approach for case 3, which is that the proxy
mapper and the name resolver both have to agree on the list of known proxy
addresses. So, for example, if the proxy mapper implementation is getting the
list of proxy addresses from a local file, the service owner would need to first
push an updated list that contains the new proxy address to all clients. Then,
once all clients have been updated, the new proxy can be added to DNS.

Note that another alternative for case 3 would be to use a gRPC-level proxy
instead of a TCP-level proxy, where the proxy natively speaks gRPC. The idea is
that the proxy accepts a gRPC request from the external client and then uses its
own client on the internal side, which does the name resolution and per-call
load balancing internally to the backend servers. The trade-off here is that
this approach requires that the gRPC-level proxy be trusted by all internal
servers, which has security implications (e.g., a successful attack on the proxy
would yield priviledged access on all internal servers).

## Implementation

### C-Core

In C-core, the HTTP CONNECT handshaker was already implemented. The following
additional changes have also been made:

-   [grpc/grpc#9383](https://github.com/grpc/grpc/pull/9383) (changes HTTP
    CONNECT handshaker to be triggered via channel args)
-   [grpc/grpc#9372](https://github.com/grpc/grpc/pull/9372) (adds proxy mapper
    hook for address rewriting)
-   [grpc/grpc#9557](https://github.com/grpc/grpc/pull/9557) (adds proxy mapper
    hook for name rewriting)

C-Core checks the following places to determine the HTTP proxy to use, stopping
at the first one that is set:

1.  `GRPC_ARG_HTTP_PROXY` channel arg
2.  `grpc_proxy environment` variable
3.  `https_proxy` environment variable
4.  `http_proxy` environment variable

If none of the above are set, then no HTTP proxy will be used.

If an HTTP proxy to be used is found, C-Core then checks the follows places to
exclude traffic destined to listed hosts from going through the proxy determined
above, again stopping at the first one that is set:

1.  `no_grpc_proxy` environment variable
2.  `no_proxy`environment variable

If none of the above are set, then the previously found HTTP proxy is used.

### Go

In Go, this functionality is being provided via a custom dialer:

-   [grpc/grpc-go#1098](https://github.com/grpc/grpc-go/pull/1098)

### Java

In Java for case 1, Java's `java.net.ProxySelector` is observed and usable
starting in v1.11.0. `ProxySelector` is generally configured with the Java
properties `-Dhttps.proxyHost` and related. grpc-java supports HTTP Basic Proxy
Authentication and credentials can be specified via the default
`java.net.Authenticator`. The experimental `GRPC_PROXY_EXP=host:port`
environment variable enabled limited proxy support starting in v1.0.3.

## Open issues (if applicable)

N/A
