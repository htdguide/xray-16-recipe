# src/xrGame/smart_cover_animation_selector.h

> Declares the bridge between the smart-cover plan and the animation layer: it owns the planner, is asked for the next clip when one ends, and reports a marker inside the current clip back to the running action.

**Needs** — [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · [`smart_cover_animation_selector_inline.h`](smart_cover_animation_selector_inline.h.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md)
**Tier floor** — T2: drives the animation layer from a plan, per clip boundary

## Purpose

Declares the surface implemented in
[`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) and
[`smart_cover_animation_selector_inline.h`](smart_cover_animation_selector_inline.h.md).

Inside a smart cover the *animation drives the simulation*, not the reverse. The creature
does not decide to move and then find a clip; the clip ends, this object is asked what to
play next, and it runs a planning cycle to answer. That inversion is why this file exists
at all and why it, rather than the planner, owns the planner's lifetime.

## Exported units

- **The class** — owns the planner and the creature's animated skeleton.
- **`initialize` / `finalize`** — bracket the planner's in-cover lifecycle.
- **`select_animation`** — the central query: what should this creature play now, and does
  the clip own the creature's movement.
- **`on_animation_end`** — told by the animation layer that a clip finished; arms the next
  planning cycle.
- **`on_mark`** relay — a marker crossed inside the playing clip is reported to the current
  action; see the implementation.
- **`modify_animation`** — rescale a newly started clip's playback speed.
- **`save` / `load`** — the planner's state through the save format.
- **`setup`** — forwarded to the planner.
- **`property_storage` / `planner`** — access for the actions.

## State

```text
RECORD animation_selector
  storage           : world state (the outer planner's)
  object            : the creature
  planner           : animation planner       # owned
  skeleton_animated : the creature's animated model
  animation         : text                    # the clip currently selected
  previous_time     : real                    # playback time at the last query
  first_time        : bool
  callback_called   : bool                    # a clip boundary is pending
```

**Invariants** — `callback_called` is the whole synchronization between the animation layer
and the plan: it is set when a clip ends and cleared when the plan has answered. Exactly
one planning cycle runs per clip.
