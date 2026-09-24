# src/xrGame/alife_simulator_header.h

> Declares the save-format version stamp, implemented in [`alife_simulator_header.cpp`](alife_simulator_header.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`alife_simulator_header_inline.h`](alife_simulator_header_inline.h.md)
**Used by** — [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_header.cpp`](alife_simulator_header.cpp.md) · [`alife_simulator_header_inline.h`](alife_simulator_header_inline.h.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSimulatorHeader`, the first layer of the simulator stack and the owner of
the save file's version field. Substance is in
[`alife_simulator_header.cpp`](alife_simulator_header.cpp.md).

It takes a configuration section at construction and ignores it, like every other alife
sub-object built by the base layer; and save and load are overridable so the multiplayer
server can stamp differently.

Exported units:

- `CALifeSimulatorHeader` — construct; starts at the current format version.
- `save` / `load` — write and verify the version chunk.
- `valid` — the non-fatal form of the check, for filtering the save list.
- `version` — the loaded file's version.
