# src/xrGame/detail_path_manager.h

> Declares the detail path builder and its geometric vocabulary; implemented in [`detail_path_manager.cpp`](detail_path_manager.cpp.md), [`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md) and [`detail_path_manager_inline.h`](detail_path_manager_inline.h.md).

**Needs** — [`restricted_object.h`](restricted_object.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`detail_path_manager_inline.h`](detail_path_manager_inline.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`PHMovementControl.cpp`](PHMovementControl.cpp.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`base_monster_think.cpp`](ai/monsters/basemonster/base_monster_think.cpp.md) · [`control_animation_base_accel.cpp`](ai/monsters/control_animation_base_accel.cpp.md) · [`control_animation_base_update.cpp`](ai/monsters/control_animation_base_update.cpp.md) · [`control_direction.cpp`](ai/monsters/control_direction.cpp.md) · [`control_direction_base.cpp`](ai/monsters/control_direction_base.cpp.md) · [`control_manager_custom.cpp`](ai/monsters/control_manager_custom.cpp.md) · [`control_movement_base.cpp`](ai/monsters/control_movement_base.cpp.md) · [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`control_path_builder_base.cpp`](ai/monsters/control_path_builder_base.cpp.md) · [`control_path_builder_base_path.cpp`](ai/monsters/control_path_builder_base_path.cpp.md) · [`control_path_builder_base_update.cpp`](ai/monsters/control_path_builder_base_update.cpp.md) · _and 21 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CDetailPathManager`, the stage that turns a list of navigation vertices into a
curve a body can actually follow. Substance is split: the lifecycle, the path queries and
the turning-circle construction in [`detail_path_manager.cpp`](detail_path_manager.cpp.md);
the whole smoothing algorithm in
[`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md); the setters and the
staleness policy in [`detail_path_manager_inline.h`](detail_path_manager_inline.h.md).

## The geometric records

These are the algorithm's vocabulary and are declared nowhere else.

```text
RECORD TravelParams               # one named way of moving
  linear_velocity        : real   # signed: negative means moving backwards
  angular_velocity       : real   # used to derive the turning radius
  real_angular_velocity  : real   # what the body is actually told to turn at

RECORD TravelParamsIndex EXTENDS TravelParams
  index : int                     # the velocity identifier this entry is known by

RECORD TravelPoint
  position  : vector2             # the plane; the vertical is added at the very end
  vertex_id : int

RECORD PathPoint EXTENDS TravelParams, TravelPoint
  direction : vector2             # unit heading at this point

RECORD CirclePoint
  center : vector2 ; radius : real ; point : vector2 ; angle : real

RECORD TrajectoryPoint EXTENDS PathPoint, CirclePoint
                                  # a point plus the turning circle it sits on

RECORD Dist                       # a candidate trajectory and its traversal time,
  index : int ; time : real       #   ordered by time
```

**Invariant** — the whole construction happens in **two dimensions**. The vertical is
assigned once, at the end, by sampling the navigation mesh under each finished point. A
rebuild that carries three dimensions through the geometry will get the turning circles
wrong on slopes.

**Invariant** — `angular_velocity` and `real_angular_velocity` are separate because the
first is a *geometry* input — it sets the turning radius the path is built for — and the
second is what the body is commanded with. They differ where a creature is allowed to turn
faster than the path assumed.

## Direction-type flags

A two-bit code saying whether the start and destination legs run forwards or backwards
(`PP`, `PN`, `NP`, `NN`). It exists because a creature may reverse, and a reversing leg's
turning circles and swept angles are mirrored.

Exported units:

- `build_path` — the entry point, protected and reached through a small set of declared
  friends rather than being public. It takes the coarse vertex list and an intermediate
  index.
- `reinit` — reset to the no-path state.
- `valid` / `valid(position)` / `failed` / `actual` / `make_inactual` — the four
  independent statuses a path can have; see
  [`detail_path_manager.cpp`](detail_path_manager.cpp.md).
- `completed` / `curr_travel_point_index` / `curr_travel_point` / `path` — where the
  follower has got to.
- `direction` / `try_get_direction` — the heading at the current point, with and without a
  fallback.
- `distance_to_target` / `location_on_path` — how much path is left, and the point a given
  distance ahead.
- the start/destination position and direction setters, the velocity masks, the minimum-time
  flag, the patrol flags and the extrapolation length — **every one of which invalidates the
  path if it changes the input**.
- `add_velocity` / `velocity` / `velocities` — the creature's movement-parameter table.

**Notes** — the geometric helpers are private and numerous, and the interesting ones are
documented in [`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md). The
friend list (the movement manager, the obstacle-aware stalker movement manager, the
poltergeist movement manager, the script entity and the builder) is the real access-control
surface: it names every subsystem allowed to drive a build.
