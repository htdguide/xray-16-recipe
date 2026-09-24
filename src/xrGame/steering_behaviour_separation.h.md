# src/xrGame/steering_behaviour_separation.h

> Declares the separation force of the abandoned rat flocking design. Never implemented.

**Needs** — [`steering_behaviour_base.h`](steering_behaviour_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an unimplemented interface case.

## Purpose

One of the three forces of the rat flocking triple that was designed and never built; see
[`steering_behaviour_base.h`](steering_behaviour_base.h.md). This one declares
**separation**: steering away from neighbours that have come too close, which is what stops
a cohering group collapsing into one point.

It has no implementation anywhere in the tree, and nothing includes it. The live steering
module implements separation as the other half of its grouping behaviour — with its own
range limit and its own falloff coefficients, which is the part that matters and is
described in [`steering_behaviour.cpp`](steering_behaviour.cpp.md). A rebuild discards this
file.

## Exported units

- construction from a rat.
- `direction()` — the separation direction. Declared, never defined.
