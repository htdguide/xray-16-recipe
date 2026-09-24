# src/xrGame/alife_update_manager.h

> Declares the top of the alife simulator: the per-frame tick, the world-scale lifecycle operations, and the script-facing verbs, implemented in [`alife_update_manager.cpp`](alife_update_manager.cpp.md).

**Needs** — [`alife_switch_manager.h`](alife_switch_manager.h.md) · [`alife_surge_manager.h`](alife_surge_manager.h.md) · [`alife_storage_manager.h`](alife_storage_manager.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`alife_simulator.cpp`](alife_simulator.cpp.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the class that *is* the alife simulator from the outside. Everything below it in
the pile — switching, surges, storage, and through those the object, graph, schedule and
spawn registries — is assembled here into one type that the rest of the game holds a
single handle to, and that the engine's scheduler drives like any other scheduled object.

The assembly is by inheritance from three managers plus the scheduled-object contract. In
a rebuild this is composition: the simulator *has* a switch pass, a surge pass, a storage
manager and a schedule slot. Nothing about the behaviour depends on the union being one
object — only the fact that every caller reaches all of it through one name does.

Substance in [`alife_update_manager.cpp`](alife_update_manager.cpp.md).

Exported units:

- `update`, `update_switch`, `update_scheduled` — one tick, and its two halves, callable
  separately because the level's load sequence needs the switch pass on its own.
- `shedule_Name`, `shedule_Scale`, `shedule_Needed`, `shedule_Update` — the scheduled-object
  contract: this object is named `alife_simulator`, is always due, and reports a fixed
  importance of one half.
- `load`, `load_game`, `new_game` — bring a world up from a save or from the spawn file.
- `change_level` — the autosave-and-restart dance that moves the actor between levels.
- `reload`, `set_process_time`, `objects_per_update`, `update_monster_factor` — the tuning
  path, and the two budgets pushed down to the registries that spend them.
- `set_switch_online`, `set_switch_offline`, `set_interactive` — per-entity vetoes on the
  online/offline transition, and the player-interaction flag.
- `jump_to_level`, `teleport_object` — move the actor, or anything, across the world.
- `add_restriction`, `remove_restriction`, `remove_all_restrictions` — maintain a
  creature's dynamic restrictor lists.
- `init_ef_storage` — put the shared evaluation storage into alife mode.
