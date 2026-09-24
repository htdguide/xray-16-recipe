# src/xrGame/alife_monster_detail_path_manager_script.cpp

> Exports the offline detail-path mover to the script layer.

**Needs** — [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a registration table

## Purpose

Binding only. It publishes `CALifeMonsterDetailPathManager` to scripts under that exact
name, so that a smart terrain's script can retarget an offline creature and then ask
whether it has arrived. The name and the method names are frozen by conformance criterion
10.

## Exported surface

**Contract** — the class is exported with no constructor (scripts only ever receive one
that the engine already owns, reached through the creature's movement manager). Methods:

- `target` — overloaded three ways: by game vertex, level vertex and position; by game
  vertex alone; and by a smart-terrain task handle.
- `speed` — overloaded as getter and setter.
- `completed`, `actual`, `failed` — the three status predicates.

**Notes** — the setters that would let a script corrupt the path (`make_inactual`,
`path`, `walked_distance`, the online/offline hooks) are deliberately *not* exported:
scripts state intent, the simulation decides how to get there. A rebuild choosing to
export more widens a frozen surface, which is the one direction that is always safe, but
it should not export less.
