# src/xrGame/visual_memory_params.h

> One tuned vision profile: the ten numbers that decide how fast a creature notices, how far it sees off-axis, and how long it keeps believing.

**Needs** — [`visual_memory_params.cpp`](visual_memory_params.cpp.md)
**Used by** — [`visual_memory_manager.cpp`](visual_memory_manager.cpp.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`visual_memory_params.cpp`](visual_memory_params.cpp.md)
**Tier floor** — T3: a plain record of tuning numbers

## Purpose

Declares the record loaded in
[`visual_memory_params.cpp`](visual_memory_params.cpp.md). It is a separate file because a
creature holds *two* of these — a free profile and a danger profile — and switches between
them whole, so the profile has to be a value the manager can hold twice rather than a block
of fields on the manager.

## State

```text
RECORD VisionParameters
  min_view_distance       : real   # multiplier on the eye range, at the frustum edge
  max_view_distance       : real   # multiplier on the eye range, on the eye axis
  visibility_threshold    : real   # accumulator value at which "noticed" flips true
  always_visible_distance : real   # reach at or below this means instant notice
  time_quant              : real   # seconds one unit of exposure is measured against
  decrease_value          : real   # drained per pass while the target is out of reach
  velocity_factor         : real   # how much the target's own motion betrays it
  transparency_threshold  : real   # surface opacity below which a ray still sees through
  luminocity_factor       : real   # exponent applied to the light falling on the target
  still_visible_time      : int    # ms a refreshed sighting keeps its "visible" bit
```

**Invariants** — the two view distances are *multipliers* on the owner's configured eye
range, not absolute distances, and `min` is the off-axis one while `max` is the on-axis
one; the naming reads backwards until you know that. `always_visible_distance` is compared
against the computed *reach*, not against the distance to the target, so it is also
expressed in the same units as the reach.

**Notes** — Only `transparency_threshold` and `still_visible_time` are meaningful for every
owner. The remaining eight drive the accumulator, and in a build that does not give monsters
stalker-grade vision they are simply never read for a monster — see
[`visual_memory_params.cpp`](visual_memory_params.cpp.md).
