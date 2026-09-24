# src/xrGame/ai/monsters/control_path_builder.h

> Declares the path resource — the creature's movement manager wearing a control-channel face — implemented in [`control_path_builder.cpp`](control_path_builder.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md) · [`movement_manager.h`](../../movement_manager.h.md)
**Used by** — [`control_direction.cpp`](control_direction.cpp.md) · [`control_direction_base.cpp`](control_direction_base.cpp.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager.h`](control_manager.h.md) · [`control_movement.cpp`](control_movement.cpp.md) · [`control_movement_base.cpp`](control_movement_base.cpp.md) · [`control_path_builder.cpp`](control_path_builder.cpp.md) · [`control_path_builder_base.cpp`](control_path_builder_base.cpp.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md) · [`control_path_builder_base_update.cpp`](control_path_builder_base_update.cpp.md) · [`poltergeist_movement.h`](poltergeist/poltergeist_movement.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlPathBuilder`, which is at once the chapter-23 movement manager for a
creature and the pure element on the path channel. Substance is in
[`control_path_builder.cpp`](control_path_builder.cpp.md).

## State

`SControlPathBuilderData` — the channel payload: enable flag, target position and mesh
vertex, path type and game-graph target, arrival orientation, minimum-time and
extrapolation preferences, the admissible and desirable gait masks, and a force-replan
flag. Described in the implementation twin.

Exported units:

- `load`, `reinit`, `update_schedule` — lifecycle; the scheduled tick is where the payload
  is applied and the path is replanned.
- `on_travel_point_change`, `on_build_path` — movement-manager hooks re-raised as bus
  events.
- `can_use_distributed_computations` — refuses to spread a search across frames while the
  player is looking at the creature.
- `is_path_end`, `is_moving_on_path`, `is_path_built` — the three predicates the rest of
  the chapter branches on.
- `valid_destination`, `valid_and_accessible`, `fix_position` — validate a target and snap
  its height onto the mesh.
- `get_node_in_radius` — a random accessible vertex between two radii.
- `find_nearest_vertex` — the expensive nearest-vertex search.
- `build_special`, `make_inactual` — private; build a straight-line path at once, and
  force the next plan to be treated as stale. `build_special` is reached through the
  control manager, which checks the caller holds the channel.

The class declares the control manager a friend so the manager can reach the two private
operations on a capturer's behalf; in a rebuild that is a second, access-checked interface
on the same object, not a language feature.
