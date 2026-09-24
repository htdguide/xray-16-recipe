# src/xrGame/detail_path_manager_space.h

> The two things the rest of the engine needs from the detail path: which smoothing style was asked for, and what one point of the finished path carries.

**Needs** — _(none beyond the math layer)_
**Used by** — [`detail_path_manager.h`](detail_path_manager.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`script_movement_action.cpp`](script_movement_action.cpp.md) · [`script_movement_action.h`](script_movement_action.h.md) · [`script_movement_action_script.cpp`](script_movement_action_script.cpp.md) · [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md)
**Tier floor** — T3: an enumeration and a record

## Purpose

A separate header so that code which only *consumes* a detail path — the movement manager,
the animation selector, the debug overlays — does not have to pull in the builder's
several-hundred-line interface. It is a deliberate and useful split.

## State

```text
ENUM DetailPathType
  smooth            # circle-line-circle trajectories honouring the turn radius
  smooth_dodge      # smooth, plus room for a sidestep
  smooth_criteria   # smooth, chosen against a cost criterion

RECORD TravelPathPoint          # one sample of the finished detail path
  position  : vector            # world space, with the vertical filled in from the
                                #   navigation mesh after the path is built
  vertex_id : int               # the navigation vertex this sample lies inside
  velocity  : int               # index into the creature's velocity table, NOT a speed
```

**Invariant** — `velocity` is an *identifier*, not a magnitude. It selects a named entry in
the creature's movement-parameter table (walk, run, crouch, each with a linear and an
angular speed), which is what lets the path record "run this stretch, turn slowly here"
without the geometry knowing anything about the creature's animation set.

**Invariant** — every point's vertex identifier must be a vertex the point actually lies
inside. The builder establishes this by stepping the navigation mesh alongside the geometry
rather than by looking the vertex up from the position afterwards, because a position on a
shared edge would otherwise resolve ambiguously.

**Notes** — the three path types all resolve to the same builder in the shipped code; see
[`detail_path_manager.cpp`](detail_path_manager.cpp.md). The enumeration is wider than the
implementation.
