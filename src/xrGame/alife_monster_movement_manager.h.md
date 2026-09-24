# src/xrGame/alife_monster_movement_manager.h

> Declares the offline creature's movement arbiter, implemented in [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md).

**Needs** — [`movement_manager_space.h`](movement_manager_space.h.md) · [`alife_monster_movement_manager_inline.h`](alife_monster_movement_manager_inline.h.md)
**Used by** — [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_monster_brain_script.cpp`](alife_monster_brain_script.cpp.md) · [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md) · [`alife_monster_movement_manager_inline.h`](alife_monster_movement_manager_inline.h.md) · [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md) · [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeMonsterMovementManager`: an owning composition of a detail mover and a
patrol-path manager, plus the mode that selects between them. Substance is in
[`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md).

The movement-mode enumeration is shared with the *online* movement manager rather than
being private to the alife side — offline and online movement name the same three modes,
which is how a creature's mode survives the transition between them.

Exported units:

- `CALifeMonsterMovementManager` — constructed against the owning holder; creates and
  owns both sub-managers.
- `detail` / `patrol` — the two sub-managers, by reference.
- `path_type` — get and set the movement mode.
- `update` — the per-tick dispatch.
- `on_switch_online` / `on_switch_offline` — forwarded to the detail mover.
- `completed` / `actual` — always true at this level.
- A script registration hook; surface in
  [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md).
