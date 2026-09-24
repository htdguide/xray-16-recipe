# src/Layers/xrRender/QueryHelper.h

> The five-call shim that hides two very different device query models behind one shape, so the occlusion-culling code never learns which backend it is running on.

**Needs** — [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__occlusion.cpp`](r__occlusion.cpp.md) · [`r__sync_point.cpp`](r__sync_point.cpp.md)
**Tier floor** — T1: it is a direct wrapper over device objects and exists only because two devices spell the same idea differently.

## Purpose

A **query** asks the graphics device a question whose answer is not available until the work it brackets has actually run: chiefly "how many fragments of this proxy geometry survived the depth test", which is what the occlusion culling in the visibility walk is built on. Both supported devices offer this, and they disagree about almost everything else: one hands out an object and takes it back for release, the other hands out an integer name; one begins and ends a query on a submission context, the other on the global state; one can answer questions of several *kinds*, the other only this one.

This file is the adapter that makes those two look alike. It is the entire content: there is no state, no policy, no allocation strategy. It is listed here because the *shape* it settles on is what the rest of the renderer is written against, and a rebuild must offer the same five operations whatever its device does.

## State

Stateless.

## The query interface

**Contract** — Five operations over an opaque query handle. A rebuild's graphics-device filling must provide all five.

```text
FUNCTION create_query(kind) -> result<QueryHandle, Error>
  # `kind` names what is being asked. Only the occlusion kind is portable:
  # the second backend rejects anything else outright, which is the honest
  # statement of what the renderer may rely on.

FUNCTION begin_query(handle)
  # opens the bracket; everything drawn until the matching end is counted

FUNCTION end_query(handle)
  # closes the bracket; the answer is not available yet

FUNCTION get_data(handle, out_buffer, out_size) -> result<Available, Pending>
  # polls for the answer. May legitimately report "not ready" — the caller
  # decides whether to wait, to stall, or to use last frame's answer.
  # `out_size` selects the width of the answer: a 64-bit slot asks for the
  # wide form, anything else for the 32-bit form.

FUNCTION release_query(handle)
```

**Invariants**

- **Only the occlusion kind is guaranteed.** The interface is parameterized by kind because one device supports several, but the second backend asserts on anything else. Any renderer code that asks for a timestamp query is therefore backend-specific by construction, and a rebuild targeting a single API should either implement the other kinds or narrow the interface to the one that is real.
- **Answers are fetched on the immediate submission context only.** Queries may be issued from a deferred-recording command list, but the result can only be read back where work actually completes. This is the one place in the shim where the multi-context design leaks: a rebuild that records rendering on worker threads must route query readback back to the single submitting thread.
- **Beginning and ending are not symmetric in what they name.** One device ends a query by naming the query; the other ends whatever query of that kind is currently open, naming only the kind. So the interface's contract is the *stricter* of the two: at most one query of a given kind may be open at a time, and it must be ended before another is begun. Nesting works on one backend and silently misbehaves on the other.
- The answer's width is selected by the size of the buffer the caller offers, not by a separate parameter. A rebuild should make this an explicit choice; inferring a format from a buffer size is the kind of implicit contract that breaks silently when a caller's type changes.

**Notes** — Fetching a result can be asked to avoid flushing pending work, at the cost of more often reporting "not ready". The engine takes the flushing default, which is the safe choice and the slower one: a poll that flushes turns a cheap check into a synchronisation point. A rebuild profiling its occlusion pass should revisit this before anything else in the file.

There is no query *pooling* here and none anywhere else in the shim's callers' shared code — each user of occlusion queries manages its own handles and its own latency policy. That is a genuine gap rather than a decision: the same "issue, wait a frame, read" dance is reimplemented by each caller.
