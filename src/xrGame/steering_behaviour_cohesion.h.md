# src/xrGame/steering_behaviour_cohesion.h

> Declares the cohesion force of the abandoned rat flocking design. Never implemented.

**Needs** — [`steering_behaviour_base.h`](steering_behaviour_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an unimplemented interface case.

## Purpose

One of the three forces of the rat flocking triple that was designed and never built; see
[`steering_behaviour_base.h`](steering_behaviour_base.h.md). This one declares **cohesion**:
steering toward the centre of the neighbours, which is what keeps a group a group.

It has no implementation anywhere in the tree, and nothing includes it. The live steering
module does implement cohesion — as one half of its grouping behaviour, described in
[`steering_behaviour.cpp`](steering_behaviour.cpp.md) — so a rebuild that wants the rule
takes it from there and discards this file.

## Exported units

- construction from a rat.
- `direction()` — the cohesion direction. Declared, never defined.
