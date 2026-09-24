# src/xrGame/alife_smart_terrain_task_script.cpp

> Exports the job destination to the script layer.

**Needs** — [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a registration table

## Purpose

Binding only. Publishes `CALifeSmartTerrainTask` under that name. Unlike most alife
bindings, this one exports **constructors**: the shipped smart-terrain scripts build
destinations themselves and hand them to the engine, which is the whole point of the type
being a small value rather than an engine-owned object.

## Exported surface

**Contract** — three constructors and three readers:

- construct from a patrol path name (point zero);
- construct from a patrol path name and a point index;
- construct from a game-graph vertex and a level-graph vertex.
- `game_vertex_id`, `level_vertex_id`, `position` — the destination triple.

**Notes** — the shared-string constructor overloads are not exported separately; scripts
pass plain strings and the binding converts. And a script-constructed task holds a
reference into the level's patrol data, so a script that stores one across a level change
holds a dangling reference — the same lifetime hazard the type has in the engine, now
reachable from Lua. A rebuild storing the path name instead of the point reference closes
it.
