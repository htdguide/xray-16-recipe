# src/xrGame/doors_actor.h

> Declares the per-creature door agent implemented in [`doors_actor.cpp`](doors_actor.cpp.md).

**Needs** — [`doors.h`](doors.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`doors_actor.cpp`](doors_actor.cpp.md) · [`doors_door.cpp`](doors_door.cpp.md) · [`doors_manager.cpp`](doors_manager.cpp.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `doors::actor` — despite the name, *not* the player: one of these exists per
creature that can operate doors, and it holds that creature's claims. Substance in
[`doors_actor.cpp`](doors_actor.cpp.md).

Exported units:

- construction from the creature, and destruction — which releases every held claim.
- `get_position` / `get_name` / `lua_game_object` — pass-throughs to the creature.
- `need_update` — whether the agent has claims that still need servicing.
- `update_doors(doors, average_speed)` — the per-frame decision: which nearby doors does my
  path go through, and in which state do I need each of them. Returns false when the creature
  must wait.
- `on_door_destroy` — drop a door that has gone away.
- `add_new_door` / `process_doors` / `revert_states` — private: admit one door, reconcile a
  held list against the current path, and release a whole list.
- `render` — debug only: draws each nearby door's two leaf positions.

**Notes** — the two held lists — doors held open and doors held closed — are kept *sorted by
handle* so that membership tests and the merge in `process_doors` are cheap. That ordering is
an invariant the implementation relies on.
