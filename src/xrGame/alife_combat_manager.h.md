# src/xrGame/alife_combat_manager.h

> Declares the off-screen combat layer of the simulation, almost all of it disabled.

**Needs** — [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_combat_manager_inline.h`](alife_combat_manager_inline.h.md)
**Used by** — [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) · [`alife_combat_manager_inline.h`](alife_combat_manager_inline.h.md) · [`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md) · [`alife_interaction_manager.h`](alife_interaction_manager.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md).
One layer of the off-screen simulation's inheritance chain; it also gives the simulation its
random-number source.

Exported units:

- **construction** from the server and the simulation's configuration section;
- **`kill_entity`** — turn a living off-screen creature into a corpse at a given graph
  vertex, attributing the kill; see the cpp twin.

Everything else the type once declared — the two combat party buffers, the combat type, the
detection, odds, attack and aftermath routines, and the relation test — is commented out.
The relation test is worth naming even so, because its rule survives elsewhere: two humans
are related by the reputation registry, and anything else is an enemy if its team differs
and neutral if it does not.

**Notes** — The inheritance is virtual, because the simulation's layers form a diamond over
one shared base. A rebuild composing the layers rather than stacking them avoids it.
