# src/xrCore/Math/fast_lc16.cpp

> Supplies the one thing the generator cannot do inline: get a seed from the world.

**Needs** — [`fast_lc16.hpp`](fast_lc16.hpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`fast_lc16.hpp`](fast_lc16.hpp.md)
**Tier floor** — T3: one call into a platform entropy source.

## Purpose

Holds the default construction and the argument-free reseed of [`fast_lc16.hpp`](fast_lc16.hpp.md), which are separate from the header only because they need a non-deterministic seed source and the header must stay free of that dependency.

## `default construction` and `seed()`

**Contract** — both draw one word from the platform's non-deterministic entropy source and feed it to the 32-bit seeding rule. Both may block briefly on first use, depending on how the host provides entropy. Neither fails: a host without real entropy falls back to whatever its library provides, and nothing here depends on the quality.

**Notes** — the entropy source is a single process-wide object shared by every generator, which is fine because it is consulted once per generator rather than once per draw. A rebuild that seeds from a clock instead will produce colliding streams on threads created in the same instant — which is exactly the case the scheduler hits, and is why an address-derived seed is offered as an alternative in the header.
