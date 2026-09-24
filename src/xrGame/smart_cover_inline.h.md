# src/xrGame/smart_cover_inline.h

> Transforms a loophole's authored local geometry into world space through the placed object's transform, and the four field reads.

**Needs** — [`smart_cover.h`](smart_cover.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a transform per call, on a planning path

## Purpose

Carries the bodies for [`smart_cover.h`](smart_cover.h.md). The load-bearing content is
one decision: loophole geometry is stored in the cover object's **local frame** and
transformed on every read, rather than baked to world space once at placement.

## Local-to-world accessors

**Contract** — `fov_position` and `position` transform a point; `fov_direction`,
`danger_fov_direction` and `enter_direction` transform a direction and renormalize it.
All read the placed object's current transform.

```text
FUNCTION fov_position(loophole) -> vector
  RETURN object.transform applied as a point to loophole.fov_position

FUNCTION fov_direction(loophole) -> vector
  d = object.transform applied as a direction to loophole.fov_direction
  RETURN normalize(d)
```

**Invariants** — the directions are renormalized after transforming even though the
authored vectors are already unit length. That is not redundant: the placed object's
transform is authored per level and nothing guarantees it is a pure rotation, so a scaled
placement would otherwise produce non-unit directions and break every arc test in
[`smart_cover.cpp`](smart_cover.cpp.md), which compares against the cosine of an angle.

**Notes** — recomputing on each read rather than caching is what makes a smart cover
attached to a *moving* object work at all. It costs a transform per query on a planning
path, which is the deliberate trade.

## `get_object` / `get_description` / `id` / `is_combat_cover`

**Contract** — field reads.

## `can_fire`

**Contract** — true when the cover is a combat cover **or** its own fire flag is set. A
combat cover always permits firing; the separate flag exists so a non-combat cover can
still allow it.
