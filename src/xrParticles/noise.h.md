# src/xrParticles/noise.h

> Declares the three-dimensional gradient noise and its two fractal summations.

**Needs** — [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md)
**Used by** — [`noise.cpp`](noise.cpp.md) · [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md)
**Tier floor** — T2: pure float arithmetic over a lookup table.

## Purpose

Declares the surface implemented in [`noise.cpp`](noise.cpp.md).

Exported units:

- **`noise3(point)`** — one octave of gradient noise, roughly in −1…1.
- **`fractalsum3(point, frequency, octaves)`** — signed sum of octaves; the one the
  turbulence action uses.
- **`turbulence3(point, frequency, octaves)`** — the same with each octave's absolute value,
  giving the creased look. Nothing in the engine calls it.

**Notes** — the table initializer is *not* declared here even though it must run before any of
these are called. Its one caller re-declares it at the call site. A rebuild should either
initialize the table lazily inside `noise3` or make the initializer part of this surface; the
split as it stands is how the coupling described in [`noise.cpp`](noise.cpp.md) stayed
invisible.
