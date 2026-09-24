# src/xrParticles/particle_manager.h

> Declares the one implementation of the module's public manager interface: two handle tables
> and a lock.

**Needs** — [`particle_actions.h`](particle_actions.h.md) · [`psystem.h`](psystem.h.md)
**Used by** — [`particle_manager.cpp`](particle_manager.cpp.md)
**Tier floor** — T2: two index-addressed tables behind a lock.

## Purpose

Declares the surface implemented in [`particle_manager.cpp`](particle_manager.cpp.md). The
interface it satisfies — and the contracts of every operation — are in
[`psystem.h`](psystem.h.md); this header adds only the private state: a table of pools, a
table of action lists, and the lock that guards both.

Exported units: the manager type itself, plus two lookups that are not part of the public
interface —

- **`effect_for(handle)`** — resolve a pool handle, failing hard on an out-of-range handle.
- **`action_list_for(handle)`** — the same for an action list handle.
