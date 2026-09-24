# src/xrGame/ai/monsters/dog/dog_state_manager.h

> Declares the dog's brain, implemented in
> [`dog_state_manager.cpp`](dog_state_manager.cpp.md).

**Needs** — [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`dog_state_manager.cpp`](dog_state_manager.cpp.md)
**Used by** — [`dog.cpp`](dog.cpp.md) · [`dog_state_manager.cpp`](dog_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the dog's brain as a specialization of the shared creature brain. Split from its body only
because C++ splits them.

## `CStateManagerDog`

- **construct** — takes the dog it drives; registers the nine global states and clears any claimed
  corpse
- **execute** — the priority selector, the territorial latch and the two hand-off rules
- **check_eat** — the dog's stricter appetite test, which also claims and locks a corpse
- **remove_links** — forward the "this object is being destroyed" notice to every registered state

Contracts are in the implementation twin.
