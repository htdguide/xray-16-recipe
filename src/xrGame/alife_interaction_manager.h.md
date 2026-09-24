# src/xrGame/alife_interaction_manager.h

> Declares the layer that joins off-screen combat and off-screen trading over one simulation state.

**Needs** — [`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md) · [`alife_combat_manager.h`](alife_combat_manager.h.md) · [`alife_communication_manager.h`](alife_communication_manager.h.md) · [`xrServerEntities/xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md)
**Used by** — [`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md) · [`alife_simulator.cpp`](alife_simulator.cpp.md) · [`alife_simulator.h`](alife_simulator.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md). Inherits both the
combat and communication layers, which inherit the simulation base virtually — so this type
is where the diamond closes and where the shared simulation state is constructed exactly
once.

Exported units:

- **construction** from the server and the simulation's configuration section.

The two `check_for_interaction` forms — one per entity, one per entity and graph vertex —
are commented out along with the rest of the meeting pass.

**Notes** — Its file header names it as the *communication* manager, as do the cpp and inline
files. All three are copies of the neighbouring file's header and none of the names matter.
