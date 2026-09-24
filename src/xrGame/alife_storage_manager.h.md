# src/xrGame/alife_storage_manager.h

> Declares the save/load layer of the alife simulator, implemented in [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md).

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_storage_manager_inline.h`](alife_storage_manager_inline.h.md)
**Used by** — [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`alife_storage_manager_inline.h`](alife_storage_manager_inline.h.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`alife_update_manager.h`](alife_update_manager.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeStorageManager`, the layer that adds saving and loading to the simulator
base. Substance is in
[`alife_storage_manager.cpp`](alife_storage_manager.cpp.md).

It keeps the configuration section name that the base was built from, because loading a
save destroys the whole simulation and rebuilds it — and rebuilding needs that section
again. That single stored string is the reason this layer exists as a layer rather than
as free functions.

Exported units:

- `CALifeStorageManager` — construct against a server and a configuration section; the
  current save name starts empty.
- `save(name, update_name)` — write a save; an empty name reuses the last one, and
  `update_name` false leaves the last name unchanged (the autosave path).
- `save(packet)` — the network request form, which flushes client state first.
- `load(name)` — replace the world with a saved one; reports failure rather than throwing
  for a missing or invalid file.

**Notes** — the layer is inherited *virtually*, because the simulator stack converges on
one base through more than one path. That is a C++ mechanism for "there is exactly one
base, however many routes reach it"; a rebuild composing rather than inheriting has the
question by construction.
