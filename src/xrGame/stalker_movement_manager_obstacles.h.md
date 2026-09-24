# src/xrGame/stalker_movement_manager_obstacles.h

> Declares the movement manager layer that makes a human walk around obstacles, wait for doors and not get stuck — implemented in [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) and [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md).

**Needs** — [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md) · [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md) · [`dynamic_obstacles_avoider.h`](dynamic_obstacles_avoider.h.md) · [`stalker_movement_manager_obstacles_inline.h`](stalker_movement_manager_obstacles_inline.h.md)
**Used by** — [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) · [`stalker_movement_manager_obstacles_inline.h`](stalker_movement_manager_obstacles_inline.h.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

The outermost layer of a human's movement pipeline. Beneath it, the base manager can find a
path across the navigation mesh; this layer is what makes that path survive a world where
things move: other creatures standing in the way, doors that are shut, an object that was
not there when the path was planned.

The idea a rebuilder needs before either implementation makes sense: **obstacles are
expressed as temporary marks on the navigation mesh, not as a cost term**. To plan around an
obstacle, this layer marks the mesh vertices the obstacle covers as forbidden, runs the
ordinary search, and unmarks them. That is why everything here comes in apply/remove pairs
and why the save-and-restore of a whole path exists — a search that fails while a border is
applied must leave both the mesh and the creature's current path exactly as it found them.

Two avoiders sit side by side and are *not* symmetric:

- the **static** avoider handles things that block a vertex outright and require the path to
  be replanned;
- the **dynamic** avoider handles things that are in the way right now and may not be by the
  time the creature reaches them, and can answer "just stop for a moment" instead of
  replanning.

Exported units:

- `stalker_movement_manager_obstacles` — the layer.
- `move_along_path` — the per-frame entry, with the door wait, the failure back-off and the
  two avoiders.
- `build_level_path` — the level search wrapped in obstacle borders, with a simulated walk of
  the result.
- `is_going_through` — does this creature's path cross a given segment, and how far along.
- `can_build_restricted_path` — would a path still exist if this obstacle were forbidden.
- `prediction_speed` — overridden to the animation's target speed.
- `remove_links` / `on_death` — teardown.
- `create_restricted_object` — supplies the obstacle-aware restrictor.
- `restricted_object` — that restrictor, in
  [`stalker_movement_manager_obstacles_inline.h`](stalker_movement_manager_obstacles_inline.h.md).
- `Load` — configuration, plus one behavioural switch on the path builder.

## State

```text
RECORD ObstaclesLayer                  # extends the base stalker movement manager
  doors_actor        : DoorsActor      # this creature's identity to the door system
  static_obstacles   : StaticAvoider
  dynamic_obstacles  : DynamicAvoider
  restricted_object  : ObstacleRestrictor
  last_dest_vertex   : int             # the destination the last search was for
  last_fail_time     : int             # when a search last failed
  failed_to_build_path : bool
  temp_path          : list<int>       # scratch for the trial search

  # the saved path, held across a search that may fail
  saved_state        : bool
  saved_level_path   : list<int>
  saved_detail_path  : list<TravelPoint>
  saved_detail_index : int
  saved_last_patrol_point : int
  saved_query        : ObstaclesQuery
```

**Invariants**
- A border is always removed before the routine that applied it returns, on every path
  including failure. The navigation mesh is process-wide and shared by every creature; a
  border left applied would forbid those vertices to everyone.
- The saved path is either fully saved or not saved at all — the flag guards every field.
