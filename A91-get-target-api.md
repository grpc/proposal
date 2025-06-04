# A100: Expose Channel Target via Public API

* Author(s): Rajat Naik
* Approver: TBD
* Status: Draft
* Updated: 2025-05-29
* Implementation Owner: Rajat Naik
* Reviewers: TBD
* Language: Python

## 1. Abstract

Introduce a new public API `get_target()` in gRPC Python that returns the channel's target string. This API provides a stable way for clients and libraries (such as Google Pub/Sub) to retrieve the underlying channel endpoint for observability, debugging, and routing.

## 2. Background

The Google Pub/Sub Python client and similar libraries depend on knowing the underlying gRPC channel target for observability, debugging, and routing decisions. In previous releases, these users accessed the target via private internals, for example:

```python
channel = client.transport.publish._channel
channel.target().decode("utf8")
```

With the migration to an interceptor-based architecture, this approach broke because the returned object was no longer a raw channel. A temporary workaround involved using another private API:

```python
channel = client.transport.publish._thunk("")._channel
channel.target().decode("utf8")
```

However, this too relies on unstable internals and risks breaking in future versions. To eliminate dependency on private structures, a stable and public method `get_target()` will be exposed directly on `grpc.Channel`.

## 3. Goals

1. Provide a simple, public API to retrieve the channel's target string.
2. Ensure the method works regardless of interceptor layers.
3. Return `None` if the channel is shut down or in an unusable state.

## 4. Guide-level Explanation

Users can now write:

```python
import grpc

channel = grpc.insecure_channel("localhost:50051")
target = channel.get_target()
# target == "localhost:50051"

# After channel.close()
channel.close()
print(channel.get_target())  # None
```

* If the channel is active (IDLE, CONNECTING, or READY), `get_target()` returns the configured target string (e.g., `"localhost:50051"`).
* If the channel is shut down or in a non-connectable state (TRANSIENT\_FAILURE or SHUTDOWN), `get_target()` returns `None`.

## 5. Detailed Design

### 5.1 API Signature

```python
def get_target(self) -> Optional[str]:
    """Returns the verbatim channel target if the channel is active, otherwise None."""
```

### 5.2 Implementation

1. Use the existing `cygrpc.Channel.check_connectivity_state(try_to_connect=False)` to retrieve the current raw connectivity state.
2. Map the returned `cygrpc` enum to `grpc.ChannelConnectivity` via `_common.CYGRPC_CONNECTIVITY_STATE_TO_CHANNEL_CONNECTIVITY`.
3. If the mapped state is not one of `{IDLE, CONNECTING, READY}`, return `None` immediately.
4. Otherwise, call `self._channel.target().decode("utf-8")`. Catch `UnicodeDecodeError` and return `None` if decoding fails.

```python
from typing import Optional
import grpc
from grpc._cython import cygrpc
from grpc.framework.common import _common

class ChannelWithTarget(grpc.Channel):
    def get_target(self) -> Optional[str]:
        """Returns the verbatim channel target"""
        cy_state = self._channel.check_connectivity_state(try_to_connect=False)
        state = _common.CYGRPC_CONNECTIVITY_STATE_TO_CHANNEL_CONNECTIVITY[cy_state]
        if state not in (
            grpc.ChannelConnectivity.IDLE,
            grpc.ChannelConnectivity.CONNECTING,
            grpc.ChannelConnectivity.READY,
        ):
            return None
        try:
            return self._channel.target().decode("utf-8")
        except UnicodeDecodeError:
            return None
```

**Notes:**

* `self._channel` is the underlying `cygrpc.Channel` object.
* Returning `None` for any non-connectable state simplifies client logic, making it clear when the channel is unavailable.

## 6. Examples

```python
import grpc
# Create a standard insecure channel
channel = grpc.insecure_channel("localhost:50051")
print(channel.get_target())  # Expected: "localhost:50051"

# After shutting down the channel
tchannel.close()
print(channel.get_target())  # Expected: None
```

## 7. Alternatives Considered

1. **Maintain Private Internals:** Continue using private attributes (`_channel` or `_thunk`) to retrieve the target.

   * Drawback: Breaks whenever internals change; no stability guarantee.
2. **Separate Helper Library:** Provide a standalone function that introspects channel internals.

   * Drawback: Still depends on implementation details; users need to import a separate package.
3. **Environment Variable / Configuration:** Require users to manually track and pass the target.

   * Drawback: Error-prone, duplicative, and cumbersome to maintain.

By adding a public method on `grpc.Channel`, users get a first-class, stable API that works regardless of future refactoring.

## 8. Rationale

* **Simplicity:** A single method call (`get_target()`) is clearer than complex workarounds.
* **Stability:** As part of the public surface, this API will receive compatibility guarantees.
* **Usability:** Libraries like Google Pub/Sub or other monitoring tools can rely on this method without custom logic.

## 9. Implementation Plan

1. **Modify `channel.py` in gRPC Python core:** Add the `get_target()` definition.
2. **Update `_common.py`:** Ensure `CYGRPC_CONNECTIVITY_STATE_TO_CHANNEL_CONNECTIVITY` mapping includes all relevant states.
3. **Add unit tests:** Cover:

   * Active channel returns the correct target.
   * Closed or failed channel returns `None`.
4. **Update documentation:** Add a note about `get_target()` in the public API docs and README.
5. **Release:** Include in the next minor version (e.g., 1.50.0).

## 10. Backwards Compatibility

* Existing code that does not call `get_target()` remains unaffected.
* Introducing a new method does not break any existing public APIs.

## 11. References

* [gRPC Python Repository](https://github.com/grpc/grpc/tree/master/src/python)
* [A6: Client Retries Design](https://github.com/grpc/proposal/blob/master/A6-client-retries.md)
* Internal Pub/Sub issue: `broken due to interceptor architecture` (Private link)
