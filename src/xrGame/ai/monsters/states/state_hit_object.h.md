# src/xrGame/ai/monsters/states/state_hit_object.h

> Declares a leaf state that shoves a nearby physics object out of the creature's way — registered by no creature, so it never runs.

**Needs** — [`state.h`](../state.h.md) · [`state_hit_object_inline.h`](state_hit_object_inline.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`state_hit_object_inline.h`](state_hit_object_inline.h.md)
**Tier floor** — T3: a declaration plus four tuning constants

## Purpose

Declares the surface implemented in
[`state_hit_object_inline.h`](state_hit_object_inline.h.md).

**This state is dead.** No creature's behaviour tree adds it, under any state identifier,
in the shipped source. It is documented because a rebuilder reading the creature layer will
find it and must be told it is not part of any creature's observable behaviour — not
because it is broken, but because nobody wired it up.

## Exported units

- **the object-shove state** — a start condition that looks for a suitable physics object,
  a per-tick execution that applies one impulse at a fixed delay after entry, and a fixed
  timeout.

Its four tuning numbers are hard-coded rather than configured: the state lasts 1000 ms, the
impulse lands 500 ms in, the acceptance cone is 30 degrees half-angle in both yaw and pitch,
and the impulse is 20 times the target's mass.

## State

```text
RECORD HitObjectState
  target  : optional<PhysicsObject>   # chosen by the start condition, valid for one activation
  hitted  : bool                      # has the single impulse already been applied
  scratch : list<Object>              # reused query result buffer
```
