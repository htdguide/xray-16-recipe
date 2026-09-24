# src/xrGame/ik_foot_collider.h

> Declares the ground test for a foot: three ray queries, the memory that suppresses repeating them, and the reach constants.

**Needs** — [`ik_collide_data.h`](ik_collide_data.h.md) · [`ik_foot_collider.cpp`](ik_foot_collider.cpp.md)
**Used by** — [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKFoot.h`](IKFoot.h.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`ik_foot_collider.cpp`](ik_foot_collider.cpp.md) · [`ik_limb_state_predict.h`](ik_limb_state_predict.h.md)
**Tier floor** — T2: a declaration plus one tuning constant

## Purpose

Declares the surface implemented in [`ik_foot_collider.cpp`](ik_foot_collider.cpp.md), plus
one constant and one small value type that carry real decisions.

## State

```text
RECORD PickQuery
  position : vector
  direction: vector          # must be unit length
  range    : real            # must be non-negative
  point    : CollidePoint    # which of the foot's three points this query is for;
                             # `none` means the query is unset

RECORD FootCollider
  previous_toe, previous_heel, previous_side : PickQuery   # last frame's three queries
  previous_result                            : CollideResult
```

**Invariants** — a query's validity is carried by its contact point being something other
than `none`. The position, direction and range of an invalid query are the most negative
representable values, which makes an unset query detectable without a separate flag and
makes using one obviously wrong rather than subtly wrong.

Two queries are equal when their point, range, position and direction all match within
tolerance. That comparison is the entire caching mechanism; see the implementation.

## `collide_dist`

```text
collide_dist = 0.5    # metres
```

**Contract** — how far *above* each foot point the ground ray starts. The ray is cast from
half a metre above the point, not from the point itself.

**Invariants** — starting above the point is what lets the test find ground the foot has
already sunk into. A ray from the foot point downward finds nothing when the animation has
pushed the foot below the floor, which is exactly the case foot placement exists to fix.

## Exported units

- `ik_pick_query` — one ray query, with validity and equality.
- `ik_foot_collider` — the test, plus the one-frame memory that skips it.
- `collide` — run the test for one foot.
