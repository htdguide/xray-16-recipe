# src/xrGame/Random.hpp

> Names the game module's one shared pseudo-random generator.

**Needs** — [`xrCore/Math/Random32.hpp`](../xrCore/Math/Random32.hpp.md)
**Used by** — [`Random.cpp`](Random.cpp.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the single 32-bit generator instance defined in [`Random.cpp`](Random.cpp.md),
so that every caller in the game module draws from the same stream.

Exported units:

- `Random32` — the shared generator.
