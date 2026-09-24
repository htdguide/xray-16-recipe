# src/xrGame/steering_behaviour_alignment.h

> Declares the alignment force of the abandoned rat flocking design. Never implemented.

**Needs** — [`steering_behaviour_base.h`](steering_behaviour_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an unimplemented interface case.

## Purpose

One of the three forces of the rat flocking triple that was designed and never built; see
[`steering_behaviour_base.h`](steering_behaviour_base.h.md) for why none of these four files
is live. This one declares **alignment**: the flocking rule that steers a creature toward
the average heading of its neighbours, so that a group converges on a common direction of
travel rather than merely staying together.

It has no implementation anywhere in the tree, and nothing includes it. A rebuild writes
nothing in its place. Note that the live steering module implements cohesion and separation
but *not* alignment, so this is the one flocking rule the engine never had in any form.

## Exported units

- construction from a rat.
- `direction()` — the alignment direction. Declared, never defined.
