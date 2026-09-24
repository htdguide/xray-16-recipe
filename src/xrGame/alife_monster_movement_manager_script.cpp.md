# src/xrGame/alife_monster_movement_manager_script.cpp

> Exports the offline movement arbiter to the script layer.

**Needs** — [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a registration table

## Purpose

Binding only. Publishes `CALifeMonsterMovementManager` under that name so a smart
terrain's script can put an offline creature on a patrol path or send it to a vertex, and
then reach through to either sub-manager.

## Exported surface

**Contract** — no constructor; scripts receive an engine-owned instance. Methods:

- `detail` and `patrol` — return the two sub-managers as handles the script may then call
  on. These are exported as free adapters rather than as the member accessors directly,
  because the binding layer must hand the script a *handle to the existing object*, not a
  copy of it; the distinction matters because a copy would let a script mutate a mover the
  simulation is not using.
- `path_type` — overloaded getter and setter over the shared movement-mode enumeration.
- `actual`, `completed` — the (always true) status predicates.

**Notes** — the ownership rule a rebuild must preserve: the script never owns these
objects and must never be able to destroy one. Reaching `detail()` from Lua and holding
the result past the creature's death is a dangling handle in the original; a rebuild with
a safer handle model removes a real hazard rather than changing behaviour.
