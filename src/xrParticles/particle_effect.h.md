# src/xrParticles/particle_effect.h

> Declares the particle pool: a fixed-capacity array with a live prefix, plus its birth and
> death notifications.

**Needs** — [`psystem.h`](psystem.h.md)
**Used by** — [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) · [`particle_effect.cpp`](particle_effect.cpp.md) · [`particle_manager.cpp`](particle_manager.cpp.md)
**Tier floor** — T2: a bounded array of value records with a live count; the T1 pressure comes
from the record's frozen layout, declared elsewhere.

## Purpose

Declares the surface implemented in [`particle_effect.cpp`](particle_effect.cpp.md).

Exported units:

- **`ParticleEffect`** — the pool: capacity, live count, the array, and the caller's birth and
  death callbacks with their opaque owner reference.
- **`ParticleEffect(max)`** — allocate a pool of the given capacity.
- **`resize(n)`** — change the capacity, growing the allocation only when it must.
- **`remove(index)`** — retire one particle.
- **`add(pos, posB, size, rot, vel, color, age, frame, flags)`** — admit one particle, or
  refuse if the pool is full.
