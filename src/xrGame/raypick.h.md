# src/xrGame/raypick.h

> Declares the script-facing ray cast: a configurable query object and the hit record scripts read back from it.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md)
**Used by** — [`level_script.cpp`](level_script.cpp.md) · [`raypick.cpp`](raypick.cpp.md)
**Tier floor** — T3: a parameter block and a translation of one hit record

## Purpose

Scripts need to ask "what is in front of me". The engine's object-space query takes an
origin, a direction, a range, a target mask and an object to ignore, and returns a hit as an
engine-side object pointer. Neither end of that is usable from script: the parameters want
to be set one at a time across several script statements, and the result must arrive as a
game object rather than as a client object.

This file declares both halves of that translation. The query is declared here and
implemented in [`raypick.cpp`](raypick.cpp.md); the result translation is declared and
implemented here because it is three assignments.

## State

```text
RECORD RayPickResult                 # the script-visible hit
  object   : optional<GameObject>    # none when the hit was static geometry
  range    : real                    # distance along the ray to the hit
  element  : int                     # which triangle, or which bone of a dynamic hit

RECORD RayPick                       # the script-visible query
  start    : vector
  direction: vector                  # NOT required to be unit length; see the notes
  range    : real
  targets  : set of target kinds     # static geometry / dynamic objects / both
  ignore   : optional<ClientObject>  # excluded from the results
  result   : RayPickResult           # last successful query's hit
```

**Invariants**

- `result` is only ever written by a successful query. A failed query leaves the previous
  hit in place, so a script that reads the accessors without checking the query's answer
  reads stale data. This is the type's one trap.
- A default-constructed query has a zero direction and zero range and will never hit
  anything; every field must be set before the first query.

## `RayPickResult.set`

**Contract** — converts one engine hit into the script-visible record. The hit's object is
resolved to its game-object facade when the hit was against an entity, and left absent
otherwise — a hit on level geometry carries no object. Range and element are copied
through unchanged.

**Notes** — `element` means different things depending on what was hit: a triangle index in
the static collision database, or a bone identifier on a dynamic object. Scripts are
expected to know which, from the target mask they asked for. A rebuild should consider
returning the two as distinct fields; the shipped scripts only read it after a
dynamic-target query, so the merge is not actually exercised.

## `RayPick` — accessors

**Contract** — one setter per parameter (origin, direction, range, target mask, ignored
object) and one getter per result field (the whole record, the object, the distance, the
element), plus a two-form construction: empty, or fully specified. The ignored-object
setter takes a game object and stores the client object behind it; passing nothing leaves
the previous value rather than clearing it.

## `RayPick.query`

**Contract** — declared here, implemented in [`raypick.cpp`](raypick.cpp.md).
