# src/xrGame/steering_behaviour_base_inline.h

> The enabled flag of the abandoned rat steering interface.

**Needs** — [`steering_behaviour_base.h`](steering_behaviour_base.h.md)
**Used by** — [`steering_behaviour_base.h`](steering_behaviour_base.h.md)
**Tier floor** — T3: a flag.

## Purpose

Two accessors for the flag on [`steering_behaviour_base.h`](steering_behaviour_base.h.md),
which is dead code — see that page. Nothing here is reachable, and a rebuild writes nothing
in its place.

## `enabled` (get and set)

**Contract** — reads and writes the flag. No validation, no notification.
