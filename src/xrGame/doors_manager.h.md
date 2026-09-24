# src/xrGame/doors_manager.h

> Declares the level-wide door registry implemented in [`doors_manager.cpp`](doors_manager.cpp.md).

**Needs** — [`doors.h`](doors.h.md) · [`quadtree.h`](quadtree.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`ai_space.cpp`](ai_space.cpp.md) · [`doors_manager.cpp`](doors_manager.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`script_game_object_use.cpp`](script_game_object_use.cpp.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `doors::manager`, the one registry of doors on the loaded level. Substance in
[`doors_manager.cpp`](doors_manager.cpp.md).

Exported units:

- construction from the level's bounding box — the spatial index is sized once and never
  resized.
- `register_door` / `unregister_door` — a physics object becomes a door and stops being one.
  The manager owns the door record.
- `actualize_doors_state` — the per-creature entry point: find the doors near one creature
  and let its agent decide what to do about them.
- `on_door_is_open` / `on_door_is_closed` — the world reporting that a door finished moving.
- `lock_door` / `unlock_door` / `is_door_locked` — script-driven locking.
- `is_door_blocked` — whether some other creature is holding the door the other way.
- `open_door` / `close_door` — private, reachable only by the per-creature agent, which is
  declared a friend.

**Notes** — the door record and the creature agent are declared but not defined here, so
callers that only need to *register* a door do not pull in the state machine.
