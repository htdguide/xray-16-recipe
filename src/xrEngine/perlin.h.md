# src/xrEngine/perlin.h

> Declares the one-, two- and three-dimensional coherent-noise generators; the substance is in [`perlin.cpp`](perlin.cpp.md).

**Needs** — [`perlin.cpp`](perlin.cpp.md)
**Used by** — [`Environment.cpp`](Environment.cpp.md) · [`Environment.h`](Environment.h.md) · [`perlin.cpp`](perlin.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`perlin.cpp`](perlin.cpp.md). The shared base holds what
all three dimensionalities have in common — the seed, the permutation table, the
lazily-initialised flag, and the three octave parameters — and each dimensionality adds its
own gradient table and sampling routine.

Exported units:

- **`CPerlinNoiseCustom`** — the shared base: seed, permutation table, ready flag, and the
  octave/frequency/amplitude parameters with their setters. Setting the octave count also
  sizes the per-octave time accumulator used by the one-dimensional continuous sampler.
- **`CPerlinNoise1D`** — one-dimensional noise. `Get` samples an absolute coordinate;
  `GetContinious` advances an internal per-octave phase by the delta since the last call.
- **`CPerlinNoise2D`** — two-dimensional noise with unit gradient vectors.
- **`CPerlinNoise3D`** — three-dimensional noise with unit gradient vectors.

The table size is 256 in every dimension and is the one number that must be reproduced
exactly for identical output; see the implementation twin.
