# src/xrGame/alife_surge_manager.h

> Declares the repopulation layer, implemented in [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md).

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md) · [`alife_surge_manager_inline.h`](alife_surge_manager_inline.h.md)
**Used by** — [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_surge_manager_inline.h`](alife_surge_manager_inline.h.md) · [`alife_update_manager.h`](alife_update_manager.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSurgeManager`, the layer of the simulator stack that turns spawn records
into entities. Substance is in
[`alife_surge_manager.cpp`](alife_surge_manager.cpp.md).

The public surface is a single protected routine — repopulate — plus two scratch lists.
Everything a caller can do with this layer is "spawn what is missing"; the rest of what
its name suggests (deciding *when* a surge happens, what a surge does to creatures) lives
in the script layer and in the update manager. A rebuild should name the layer for what it
does.

Like the storage manager it is inherited virtually, so that the several layers of the
simulator stack converge on one base.

Exported units:

- `CALifeSurgeManager` — construct against a server and a configuration section, both
  merely forwarded.
- `spawn_new_objects` — repopulate.
