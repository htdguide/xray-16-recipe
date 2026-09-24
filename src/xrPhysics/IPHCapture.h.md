# src/xrPhysics/IPHCapture.h

> The published handle on a character's grip of a physical object — create it,
> ask whether it failed, tell it a participant died, release it.

**Needs** — [`PHCapture.h`](PHCapture.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · [`PHCapture.h`](PHCapture.h.md)
**Tier floor** — T3: a four-call handle.

## Purpose

Some creatures pick physical objects up — pull them in and hold them against a bone. The
mechanism is a constrained two-body problem and belongs to physics; the decision to grab
belongs to the creature's brain. This header is the whole boundary between the two, and it
is deliberately tiny: the game asks for a capture, checks whether it took, and later lets go.

The substance is in [`PHCapture.cpp`](PHCapture.cpp.md) and
[`PHCaptureInit.cpp`](PHCaptureInit.cpp.md).

## `IPHCapture`

**Contract** — `Failed` reports that the capture never established or has since ended, which
is how the caller knows to stop trying; `Release` lets go voluntarily; `RemoveConnection`
severs the capture when one of the two participants is being destroyed. All three are safe
to call repeatedly.

## capture creation and destruction

**Contract** — two ways to ask for a capture, which differ only in how the target part is
chosen:

- **By proximity** — the target object is given, and the nearest of its bodies to the
  capturing bone is picked, optionally filtered by a caller-supplied predicate (so a creature
  can refuse to grab, say, a part it is standing on).
- **By named part** — the caller already knows which bone of the target it wants.

Both return a handle even when the capture cannot be made; the caller discovers that through
`Failed` rather than through a null result. Destruction takes the handle by mutable
reference and clears it, because a capture that outlives either participant is the failure
mode this whole path exists to avoid.

**Notes** — the capture parameters (reach, force, time limit) are not arguments: they are
read from the *capturing creature's* skeleton configuration, so a creature with no `capture`
section in its data simply cannot capture. That is the data-driven switch for the whole
feature.
