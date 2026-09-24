# src/xrGame/ai/monsters/fracture/fracture_state_manager.h

> Declares the fracture's brain, implemented in
> [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md).

**Needs** — [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md)
**Used by** — [`fracture.cpp`](fracture.cpp.md) · [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the fracture's brain as a specialization of the shared creature brain. Split from its body
only because C++ splits them.

## `CStateManagerFracture`

- **construct** — takes the fracture it drives; registers the six generic global states
- **execute** — the reduced solitary priority selector
- **remove_links** — forward the "this object is being destroyed" notice to every registered state

Contracts are in the implementation twin.
