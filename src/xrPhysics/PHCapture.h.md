# src/xrPhysics/PHCapture.h

> Declares the object that holds a physical thing against a creature's bone — the
> internal side of the grip whose public face is [`IPHCapture.h`](IPHCapture.h.md).

**Needs** — [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`IPHCapture.h`](IPHCapture.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`IPHCapture.h`](IPHCapture.h.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md)
**Tier floor** — T1: it owns a body, two joints and its own solver island, and reads joint
feedback between the solve and the integration.

## Purpose

Declares the type whose substance is split across
[`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) (construction, parameters, release) and
[`PHCapture.cpp`](PHCapture.cpp.md) (the per-step state machine). The split is by lifecycle
phase rather than by concern and is arbitrary; a rebuild should merge them.

A capture is simultaneously three things: a per-step updated object, a holder of joints, and
the published handle the game layer sees. It also carries **its own island**, which is
unusual — see [`PHCapture.cpp`](PHCapture.cpp.md) for why.

## exported units

- **two constructors** — by proximity (pick the nearest body of the target to the capturing
  bone, optionally filtered) or by named part. Both leave the capture in the free state
  rather than failing, if the grab is not possible.
- **`Failed`** — true while the capture is in the free state, which covers "never
  established" and "finished" alike.
- **`Release`** — let go voluntarily.
- **`RemoveConnection`** — sever the capture because a named participant is being destroyed.

The rest of the surface is the per-step machinery: the state machine's four live states, the
per-step hooks, the contact callback that keeps capturer and captured from colliding, and
the notification that a shell is going away.

**Notes** — the state enumeration is the type's real shape: *pulling* (the object is being
drawn in), *captured* (it is held), *released* (let go, but still overlapping), *free* (done)
and a failed state that the shipped code never enters. A rebuild should delete the unused
one; leaving it invites a reader to look for the transition into it.
