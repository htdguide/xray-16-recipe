# src/xrGame/stalker_velocity_collection.h

> Declares the per-section movement speed table.

**Needs** — [`stalker_velocity_collection.cpp`](stalker_velocity_collection.cpp.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`stalker_velocity_collection_inline.h`](stalker_velocity_collection_inline.h.md)
**Used by** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_velocity_collection.cpp`](stalker_velocity_collection.cpp.md) · [`stalker_velocity_collection_inline.h`](stalker_velocity_collection_inline.h.md) · [`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md) · [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md)
**Tier floor** — T2: a small fixed-shape table.

## Purpose

Declares the surface implemented in
[`stalker_velocity_collection.cpp`](stalker_velocity_collection.cpp.md) — the load — and in
[`stalker_velocity_collection_inline.h`](stalker_velocity_collection_inline.h.md) — the
lookup, which carries the rules about which postures are legal in which mental state.

The posture enumerations it indexes by (mental state, body state, movement type, movement
direction) are the creature-wide vocabulary shared with every other movement consumer, not
local to this file.

## Exported units

- construction from a configuration section name — loads nineteen speeds.
- `velocity(mental state, body state, movement type, movement direction)` — the lookup.
