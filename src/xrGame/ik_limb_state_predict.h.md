# src/xrGame/ik_limb_state_predict.h

> What one limb knows about its *next* footfall, so the body can start lowering before the foot lands.

**Needs** — [`ik_foot_collider.h`](ik_foot_collider.h.md)
**Used by** — [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md)
**Tier floor** — T2: a record

## Purpose

Aligning a foot to a step is not enough on its own: if the whole body does not lower to meet
it, the leg reaches its limit and the placement fails. So the system looks ahead — how long
until this foot next lands, and how far the body must shift when it does — and begins the
shift early. This record holds that look-ahead for one limb.

## State

```text
RECORD LimbPrediction
  time_to_footstep : real = infinity   # seconds until this limb's next planted span begins
  footstep_shift   : real = 0          # the vertical body shift that footfall will demand
  collider         : FootCollider      # a SEPARATE ground tester for the predicted position
```

**Invariants** — the prediction owns its **own** ground tester, distinct from the one used for
the current frame's placement. It must: the prediction tests the ground at a position the foot
has not reached yet, and the tester caches its last query. Sharing one would have the
prediction's query evict the current frame's cache every frame, defeating the caching that
makes foot placement affordable.

The initial time is infinity rather than zero, so a limb with no known upcoming footfall never
triggers an early shift.

The shift is a signed vertical distance, consumed by the body-shift smoother in
[`ik_object_shift.cpp`](ik_object_shift.cpp.md), which spreads it over the predicted time.
