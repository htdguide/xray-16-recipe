# src/xrGame/ai/monsters/flesh/flesh_state_manager.h

> Declares the flesh's brain, implemented in
> [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md).

**Needs** — [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md)
**Used by** — [`flesh.cpp`](flesh.cpp.md) · [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the flesh's brain as a specialization of the shared creature brain. Split from its body only
because C++ splits them.

## `CStateManagerFlesh`

- **construct** — takes the flesh it drives; registers the nine generic global states
- **execute** — the reference solitary priority selector
- **remove_links** — forward the "this object is being destroyed" notice to every registered state

Contracts are in the implementation twin.
