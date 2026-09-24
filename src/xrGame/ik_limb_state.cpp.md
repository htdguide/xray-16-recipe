# src/xrGame/ik_limb_state.cpp

> Converts a saved limb placement between the two bones it can be expressed against.

**Needs** — [`ik_limb_state.h`](ik_limb_state.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md)
**Used by** — reached through its declarations in [`ik_limb_state.h`](ik_limb_state.h.md); callers name that, not this file.
**Tier floor** — T2: transform composition

## Purpose

A limb's placement can be expressed against either of its two end bones — the ankle or the
toe — and which one is correct depends on how the foot is contacting the ground, so it can
change from frame to frame. This file is the conversion between them, applied on every read
of a saved placement. The rules that decide *when* a conversion is needed are in
[`ik_limb_state.h`](ik_limb_state.h.md).

## State

`Stateless.`

## `set_limb`

**Contract** — binds the state to a limb and adopts that limb's current reference bone as the
state's own. Called when the state is first attached, so that the first read needs no
conversion.

## `to_ref_bone`

**Contract** — converts a placement from the bone the state stored it against to the bone the
limb currently uses. A no-operation when they agree.

```text
FUNCTION to_ref_bone(placement)
  IF state.ref_bone = limb.ref_bone THEN RETURN placement unchanged

  conversion = state.b2tob3              # the stored transform between the two end bones
  IF state.ref_bone is the FARTHER bone THEN conversion = inverse(conversion)
  # no other pair is possible; anything else is a programming error

  RETURN placement composed with conversion
```

**Invariants** — only two bones and therefore only two directions exist. The stored transform
goes one way and is inverted for the other; there is no general bone-to-bone path. That
restriction is deliberate and is why the transform can be a single cached matrix instead of a
walk up the skeleton, which is what the source's disabled general-case line would have done.

The transform is **cached in the state at solve time**, not recomputed here. It is a property
of the current pose, so recomputing it on read would use a later pose than the placement was
computed against.

## `anim_pos`, `goal`, `blend_to`

**Contract** — each reads the corresponding saved placement and converts it. The goal and
blend target carry their collision classification through the conversion unchanged: the
classification describes *how* the placement was constrained, which does not depend on which
bone it is expressed against.

## `pick`

**Contract** — the ground-search direction, converted between bone frames. Unlike the
placements, this one is transformed as a *point* rather than composed as a transform — so it
picks up the conversion's translation as well as its rotation.

**Notes** — that is very likely wrong: a direction should be rotated, not translated, and the
two end bones of a foot are separated by the foot's length. In practice the direction is
almost always the world vertical and the reference bone almost never changes mid-stride, so
the error is rarely reached. A rebuild should transform it as a direction and should expect a
small difference from the original on the frames where the reference bone does change.
