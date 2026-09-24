# src/xrGame/raypick.cpp

> Runs a script's ray cast against the loaded level and keeps the nearest hit.

**Needs** — [`raypick.h`](raypick.h.md) · [`Level.h`](Level.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`raypick.h`](raypick.h.md); callers name that, not this file.
**Tier floor** — T2: delegates one query to the collision database

## Purpose

The single point where script-authored ray casts enter the engine's object space. Everything
interesting — the acceleration structure, the dynamic-object broad phase, the bone-level
refinement — belongs to the level's object space; this file is the adapter that hands it the
query object's fields and translates the answer back.

## State

`Stateless.` The parameters and the last hit live in the query object declared in
[`raypick.h`](raypick.h.md).

## Construction

**Contract** — two forms. The empty form zeroes the origin, the direction and the range,
clears the target mask and ignores nothing; such a query cannot hit anything until every
field is set. The full form takes origin, direction, range, target mask and a game object to
ignore, resolving that object to the client object the query actually excludes. Absent
parameters do not fail; they leave the query unable to hit.

## `query`

**Contract** — casts the ray against the loaded level's object space and answers whether
anything was hit. On a hit, the result record is overwritten with the nearest intersection
and its object resolved to a game object; on a miss the result is left untouched. Blocks for
the duration of the cast. Does not allocate. Requires a loaded level — there is no query
outside one.

```text
FUNCTION query() -> bool
  hit = level.object_space.ray_pick(start, direction, range, targets, ignore)
  IF hit EXISTS THEN
    result.set(hit)
    RETURN true
  RETURN false
```

**Invariants** — the query asks for the *nearest* hit, not all hits, so the result is a
single record rather than a list. A rebuild wanting a script-visible all-hits cast must add
a second entry point; widening this one changes the result shape scripts already depend on.

**Notes**

- The direction is passed through to the collision database unnormalized. The database
  treats the range as a parameter along the given direction, so a script that supplies a
  non-unit direction silently scales its own range. Nothing in the engine normalizes it and
  nothing documents it; a rebuild should normalize on the way in and say so.
- The ignored object excludes exactly one object, not a set. There is no way for a script to
  ignore a weapon and its owner in one cast — the usual workaround in the shipped scripts is
  to start the ray past the ignored geometry.
