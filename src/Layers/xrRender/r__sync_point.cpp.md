# src/Layers/xrRender/r__sync_point.cpp

> A rotating set of device fences, one per graphics processor, so the frame loop can wait for work submitted a fixed number of frames ago rather than for the frame it just submitted.

**Needs** — [`r__sync_point.h`](r__sync_point.h.md) · [`QueryHelper.h`](QueryHelper.h.md) · [`HWCaps.h`](HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__sync_point.h`](r__sync_point.h.md)
**Tier floor** — T1: device synchronisation primitives with explicit lifetimes and a spin-with-timeout wait.

## Purpose

The processor must not run arbitrarily far ahead of the graphics device — input latency grows and memory fills with queued frames — but it must not wait for the device either, or the two never overlap. The standard answer is a fence a fixed number of frames back, and this is it.

The number of frames back is the number of graphics processors the device reports. On a single-processor system that is one and the wait is for the previous frame; on a multiple-processor configuration each processor works on its own frame and the wait must span all of them.

## State

```text
RECORD SyncPoint
  fences  : list<device fence> indexed 0 .. processor_count - 1
  current : int                # which fence the next wait will use
```

Invariants:

- `current` advances by one modulo the processor count once per frame, in `end`. A wait therefore targets the fence placed `processor_count` frames ago.
- The fence at index zero is signalled immediately at creation on the backend that needs it, so the very first wait has something to wait on rather than an unsignalled object.

## `create` / `destroy`

**Contract** — Make and release one fence per graphics processor. On the backend whose fences are created per signal rather than up front, both are empty — there is nothing to pre-create.

## `wait`

**Contract** — Block until the device has finished everything submitted before the current fence, or until a timeout. Returns whether the device actually caught up. Takes the sleep interval to use between polls and the timeout.

The two backends reach this differently and the difference is real:

- On the backend with explicit fence objects, the fence is a query whose result becomes available when the device reaches it. The wait is a poll loop: ask, and if the answer is not ready, yield to another thread or sleep for the configured interval; give up at the timeout and report failure.
- On the backend with a native fence primitive, the fence is *created here* — placed into the command stream at the point of the wait — and then waited on with a flush hint and a timeout, in one call. Three outcomes are distinguished: already signalled and satisfied both count as success; a timeout is reported as a failure but is deliberately not treated as an error, because a device that is merely slow must not stop the frame loop; a wait failure is logged and is a genuine fault.

**Invariants** — A timeout resolves to "did not catch up", and the caller's correct response is to carry on rather than to retry. Blocking indefinitely on a device fence is how a hung device takes the whole process with it.

**Notes** — That one backend creates its fence inside `wait` and the other creates all of them in `create` is why the abstraction is "a sync point" rather than "a fence": the object being waited on is not the same kind of thing in the two cases, and only the *when* is shared.

## `end`

**Contract** — Advance to the next fence and place it. On the backend with explicit fence objects, placing means ending the query at the current point in the command stream; on the other, the fence just advanced past is deleted, since a new one is created at the next wait.

The advance happens before the placement, so `wait` and `end` operate on the same index within a frame and the rotation is what separates frames.
