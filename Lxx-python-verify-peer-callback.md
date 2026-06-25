Python: Expose `verify_peer_callback` for Custom TLS Certificate Verification
----
* Author(s): Abdelrahman Ibrahim
* Approver: TBD
* Status: Draft
* Implemented in: Python
* Last updated: 2026-04-16
* Discussion at: TBD

## Abstract

Add an optional `verify_peer_callback` parameter to
`grpc.ssl_channel_credentials()` and `grpc.ssl_server_credentials()`,
backed by the existing C-core `grpc_tls_certificate_verifier_external`
API. The callback runs synchronously on the same TLS connection that
will carry gRPC traffic, enabling use cases such as OCSP revocation
checking, certificate pinning, and custom certificate-policy checks
without a preflight TLS connection.

## Background

gRPC Python currently has no public way to run custom peer verification
during the TLS handshake, even though C-core has supported this via
`grpc_tls_certificate_verifier_external` since gRFC #205 landed. Today
Python users who need custom verification (e.g. for OCSP revocation
checking) open a separate TLS connection to fetch the peer certificate
before creating the gRPC channel, which leaves a time-of-check-to-time-
of-use gap between the certificate that is validated and the certificate
that actually protects the traffic.

The request has been open for years:

* [#10721](https://github.com/grpc/grpc/issues/10721) (2017) — original
  request; jboeuf agreed the feature should add extra validation, not
  bypass normal HTTPS checks.
* [PR #12656](https://github.com/grpc/grpc/pull/12656) (2017–2019) —
  earlier Python/Ruby attempt on the older `verify_peer_options` path.
  Stalled on review feedback requesting async callbacks and raising
  interpreter-lock concerns. Closed unmerged.
* [#19845](https://github.com/grpc/grpc/issues/19845) (2019) —
  yihuazhang clarified that in C-core the verify callback runs after
  the TLS handshake succeeds and does not replace it.
* [#32635](https://github.com/grpc/grpc/issues/32635) (2023, open,
  `help wanted`) — gtcooke94 pointed contributors at
  `grpc_tls_certificate_verifier_external` as the C-core API to use;
  maintainers asked for a gRFC first.

This proposal uses that exact API.

## Proposal

### API

```python
def ssl_channel_credentials(
    root_certificates=None,
    private_key=None,
    certificate_chain=None,
    verify_peer_callback=None,  # NEW, optional
):

def ssl_server_credentials(
    private_key_certificate_chain_pairs,
    root_certificates=None,
    require_client_auth=False,
    verify_peer_callback=None,  # NEW, optional
):
```

### Callback contract

```python
def verify_peer_callback(target_name: str, peer_pem: str) -> None
```

* Called after the TLS handshake and underlying certificate verification
  succeed, on the same connection that will carry traffic.
* Client side: `target_name` is the server hostname (or
  `grpc.ssl_target_name_override` if set); `peer_pem` is the server
  leaf certificate.
* Server side: `target_name` is `""`; `peer_pem` is the client leaf
  certificate (meaningful only when `require_client_auth=True`).
* Raising any exception rejects the peer and fails the handshake.
* Returning normally accepts the peer; the return value is ignored.
* `verify_peer_callback=None` preserves existing behavior exactly.
* The parameter will be documented as `EXPERIMENTAL` in the initial
  release, matching gRPC Python's convention for new APIs backed by
  C-core experimental functions.

### Synchronous only, initially

`grpc_tls_certificate_verifier_external` supports both sync and async
modes. This proposal starts with sync (`return non-zero`) because:

* It is the smallest Python surface area.
* It avoids exposing completion handles and cross-thread callback
  reentry in the first version — the complexity that blocked PR #12656.
* Async can be added later without breaking the sync API.

### Leaf certificate only, initially

The callback exposes `peer_pem` (the leaf). `peer_cert_full_chain` is
not exposed in this version — it can be added later as an optional
third argument if real use cases require it.

## Rationale

### Reuse existing credential factories

Adding a parameter to `ssl_channel_credentials()` and
`ssl_server_credentials()` keeps the change additive and avoids a new
public credentials class. The client-side Python implementation
already uses the newer `grpc_tls_credentials_create` path after the
migration in commit `4cb3850cec`.

### Hostname verification is preserved

Setting a custom verifier replaces the default
`HostNameCertificateVerifier` that C-core installs when no verifier is
provided (`tls_credentials.cc:82-89`). However, hostname checking also
runs independently via `check_call_host` in the TLS channel security
connector on every RPC. `check_call_host` defaults to `true`
(`grpc_tls_credentials_options.h:133`) and this proposal does not
disable it. Hostname verification remains in effect when
`verify_peer_callback` is set — the callback is an additional hook,
not a replacement for standard TLS or hostname verification.

### Interpreter-lock concern from PR #12656

apolcyn's concern in PR #12656 was Ruby-specific — Ruby's single-
threaded event loop for gRPC could stall if a blocking callback ran on
a C-core thread. Python's gRPC uses a thread pool, and the trampoline
acquires the Python GIL with `with gil:` only for the duration of the
callback. CPython releases the GIL during blocking I/O (socket waits
for OCSP, etc.), so other Python threads are not starved. On the
server, a slow callback blocks the handshake for that connection only;
other handshakes proceed on other gRPC handshake threads.

### Server-side: two paths, preserve existing behavior

When `verify_peer_callback=None`, the server continues to use
`grpc_ssl_server_credentials_create_with_options` so
`transport_security_type` remains `"ssl"`. When a callback is
provided, the server switches to `grpc_tls_server_credentials_create`
because the older server API does not support custom verifiers.

## Implementation

All changes are in `src/python/grpcio/`. No C or C++ changes required.

1. `grpc/_cython/_cygrpc/grpc.pxi` — Cython `cdef extern` declarations
   for `grpc_tls_certificate_verifier_external`,
   `grpc_tls_custom_verification_check_request`, and related C-core
   functions already present in `include/grpc/credentials.h`.
2. `grpc/_cython/_cygrpc/credentials.pxd.pxi` — add `_verify_peer_callback`
   field to `SSLChannelCredentials` and `ServerCredentials` to keep the
   callback alive for the credential lifetime.
3. `grpc/_cython/_cygrpc/credentials.pyx.pxi` — add a `noexcept nogil`
   Cython trampoline that acquires the GIL, extracts `target_name` and
   `peer_pem` from the C-core request, invokes the Python callback,
   and translates exceptions to synchronous verifier failure. Wire the
   verifier into both `SSLChannelCredentials.c()` and the server
   credentials factory.
4. `grpc/__init__.py` — add the optional parameter and docstring.

Tests cover: client accept, client reject, client target/PEM capture,
server accept with client auth, server reject, server client-cert
capture, `None` preserves behavior, and hostname verification still
fails on mismatch.

## Open Questions

1. Should `peer_cert_full_chain` be exposed as an optional third
   callback argument in a future revision?
2. Should async verifier support be added once there is evidence of
   real-world need?
