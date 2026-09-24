# src/xrGame/detail_path_manager_inline.h

> The detail path's input setters, each of which is also the staleness test, plus the completion rule and the velocity table.

**Needs** — [`detail_path_manager.h`](detail_path_manager.h.md)
**Used by** — [`detail_path_manager.h`](detail_path_manager.h.md)
**Tier floor** — T3: field access with a comparison

## Purpose

This looks like a file of accessors and is in fact where the rebuild policy lives. **Every
setter that changes an input clears the actuality flag**, and that is the only mechanism by
which the path is ever rebuilt. Nothing polls, nothing subscribes; the movement manager sets
what it wants each frame and the path notices it changed.

## State

Adds nothing.

## The invalidating setters

**Contract** — start position, start direction, destination position, destination direction,
path type, velocity mask, desirable mask, minimum-time flag, destination-orientation flag,
patrol flag, extrapolation length. Each stores the new value and conjoins the actuality flag
with "the new value equals the old". So actuality can only ever be *lost* between builds,
never regained by setting a value back — which is correct, because a build is what grants it.

**Invariants** — the comparisons are *approximate* for the geometric inputs and exact for
the flags and masks. The destination position uses a **ten-centimetre** tolerance, much
looser than the engine's usual epsilon, and directions and the extrapolation length use the
default similarity. Ten centimetres is a deliberate hysteresis: a target that jitters — a
creature following a walking actor — would otherwise force a rebuild of a several-hundred
point path every frame.

**Invariants** — the start position and start direction setters do **not** invalidate. The
creature moves along its own path constantly; treating that as a changed input would rebuild
forever. The path is rebuilt when the *goal* changes, not when the follower advances.

**Invariants** — the destination setter records a *corrected* destination whenever the
destination genuinely moved. That copy is what `valid` later compares the path's endpoint
against; see [`detail_path_manager.cpp`](detail_path_manager.cpp.md).

**Notes** — the destination setter also asserts that the new destination is accessible under
the object's current restrictors, with a message naming the real failure mode: restrictions
changed after the destination was chosen. It is an assertion, so in a shipping build an
inaccessible destination silently produces a path that cannot be followed.

## `completed`

**Contract** — whether the follower has reached the end. Three cases, and the third is the
reason the function is not a comparison against the path size:

```text
FUNCTION completed(position, real_completion, travel_point_index) -> bool
  IF the path is empty THEN RETURN true          # nothing to walk
  IF real_completion OR NOT patrol_path THEN
    RETURN travel_point_index == last index of path
  RETURN travel_point_index >= last_patrol_point
```

**Invariants** — a patrol path extends past its destination by the extrapolation length, so
it has two ends: the *patrol* end, where the creature has arrived, and the *real* end, out in
the extrapolated tail. Asking with real completion gets the second. The tail exists so the
creature keeps walking through its waypoint instead of stopping dead on it and having to
accelerate again.

**Invariants** — the position argument is accepted and unused in every form. Completion is
decided by index, not by proximity, because the follower owns the index and advances it under
its own rules.

## `velocity` · `add_velocity` · `velocities`

**Contract** — the creature's movement-parameter table, keyed by velocity identifier.
Looking up an identifier that was never added is a contract violation, not a miss: the
builder writes identifiers into every path point and a missing entry means the path and the
table disagree.

**Notes** — insertion does not replace an existing entry, so re-adding an identifier with new
parameters silently keeps the old ones. Creatures add their velocities once at initialization,
so this has never bitten, but it makes runtime retuning impossible.

## `check_mask`

**Contract** — true when every bit of the test value is present in the mask. Note that it is
an **all-bits** test, not an intersection test, and the "test value" is a velocity identifier.
Velocity identifiers are therefore *bit patterns*, not ordinals, and a creature's velocity
set is arranged so that masking selects meaningful subsets — walk-only, any-forward, and so
on.

## `adjust_point`

**Contract** — the point at a given bearing and distance from a centre. The bearing is
measured with the world's convention (negative sine on the first axis, cosine on the second),
which must match the one the heading extraction uses or every turning circle comes out
mirrored.

## `curr_travel_point_index` · `curr_travel_point` · `path`

**Contract** — the follower's index, the point it names, and the point list itself, exposed
both read-only and writable. The index is required to be in range; reading it on an empty
path is a contract violation rather than a defined failure.

**Notes** — the writable path exposure exists so the obstacle-aware movement managers can
splice a detour into a built path. A rebuild should give that an explicit operation; handing
out the backing list means the actuality flag no longer means what it says.

## `distance_to_target`

**Contract** — the cached remaining length, recomputing on demand when the cache is stale.
See [`detail_path_manager.cpp`](detail_path_manager.cpp.md).

## The plain accessors

**Contract** — start and destination position and direction, the two masks, the flags, the
build timestamp, the destination vertex, the last patrol point (readable and writable), and —
in debug builds only — the key-point list the smoothing pass worked from, for the overlay
that draws it.
