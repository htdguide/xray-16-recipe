# src/xrGame/alife_time_manager.h

> Declares the game clock, implemented in [`alife_time_manager.cpp`](alife_time_manager.cpp.md) and [`alife_time_manager_inline.h`](alife_time_manager_inline.h.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md) · [`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_time_manager.cpp`](alife_time_manager.cpp.md) · [`alife_time_manager_inline.h`](alife_time_manager_inline.h.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`game_cl_single.cpp`](game_cl_single.cpp.md) · [`game_sv_single.cpp`](game_sv_single.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeTimeManager`. Loading, saving and initialization are in
[`alife_time_manager.cpp`](alife_time_manager.cpp.md); the derivation and the mutators are
in [`alife_time_manager_inline.h`](alife_time_manager_inline.h.md).

The type is small and its whole surface is about one derived value, so a rebuild should
read the two files as one.

Exported units:

- `CALifeTimeManager` — construct from a configuration section, which initializes the
  calendar.
- `init` — re-initialize from a section; separate from construction so that a new game can
  reset the clock without rebuilding the object.
- `game_time` — the current calendar value.
- `start_game_time` — the authored start.
- `time_factor` / `normal_time_factor` — the current and default speeds.
- `set_time_factor`, `set_game_time_factor`, `change_game_time` — the three mutations.
- `save` / `load` — the clock's chunk of a saved game; overridable.
