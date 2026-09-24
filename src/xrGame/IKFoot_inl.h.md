# src/xrGame/IKFoot_inl.h

> The foot's vector accessors: hand out the toe, heel and sole normal in the reference bone's space, converting through the bind-pose relation between the two foot bones.

**Needs** — [`IKFoot.h`](IKFoot.h.md)
**Used by** — [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKFoot.h`](IKFoot.h.md)
**Tier floor** — T2: one bind-pose transform, inlined because it is on the per-frame path of every animated character's feet

## Purpose

Split out of [`IKFoot.h`](IKFoot.h.md) only so the declaration reads as a surface; in a
rebuild these belong with the class. What they *decide* is worth stating, because it is
the rule that keeps the foot's geometry consistent: a vector belonging to the foot is
stored once, in whichever bone's space it was derived in, and is converted on demand to
whichever bone is currently the reference.

The conversion is through the **bind-pose** relation between the ankle and the toe, never
the current pose, because these vectors are fixed in the foot's shape and must not follow
the animation. Frame conversions, which must follow the animation, are the separate
current-pose ones in [`IKFoot.cpp`](IKFoot.cpp.md).

## `get_local_vector`

**Contract** — convert a stored vector into a named bone's space. Same bone: copy. Ankle
wanted, toe stored: apply the bind relation. Toe wanted, ankle stored: apply its inverse.
Anything else is a programming error, since only two bones exist.

```text
FUNCTION local_vector_in(bone, stored) -> vec3
  IF bone == stored.bone           THEN RETURN stored.v
  IF bone is the ankle AND stored is in the toe THEN RETURN bind_ankle_to_toe * stored.v
  IF bone is the toe AND stored is in the ankle THEN RETURN inverse(bind_ankle_to_toe) * stored.v
  FAIL WITH "a foot has only two bones"
```

**Notes** — the inverse is recomputed at every call rather than cached. On the per-frame
path of every foot of every visible character that is not free; a rebuild should store
both directions.

## `ToePosition` / `FootNormal`

**Contract** — the toe contact point and the sole's facing, converted into the reference
bone's space.

## `HeelPosition`

**Contract** — the heel contact point, returned **without conversion**, in the ankle's
space where it was derived.

**Notes** — the conversion is present in the source and commented out, and the toe
accessor's direct-copy version is commented out beside it. The two accessors are therefore
deliberately asymmetric and every caller must know it: the heel comes back in the ankle's
space and is converted at the call site. A rebuild should make both consistent, but must
then audit the call sites, because the placement code in
[`IKFoot.cpp`](IKFoot.cpp.md) relies on the current behaviour.
