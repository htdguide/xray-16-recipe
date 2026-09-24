# src/xrGame/alife_switch_manager_inline.h

> Where the online and offline radii come from: one authored distance and one hysteresis fraction.

**Needs** — [`alife_switch_manager.h`](alife_switch_manager.h.md)
**Used by** — [`alife_switch_manager.h`](alife_switch_manager.h.md)
**Tier floor** — T2: two derived reals

## Purpose

Small, and one of the few inline files in this chapter that holds a decision rather than
an accessor. Substance of the layer is in
[`alife_switch_manager.cpp`](alife_switch_manager.cpp.md).

## Construction

**Contract** — reads the nominal switch distance and the hysteresis factor from a
configuration section, derives the two working radii, and seeds the layer's random stream
from the processor's cycle counter.

```text
FUNCTION construct(server, section)
  switch_distance = configuration(section, "switch_distance")
  switch_factor   = configuration(section, "switch_factor")
  set_switch_distance(switch_distance)          # derives the pair
  seed random from the cycle counter
```

## `set_switch_distance` — the derivation

**Contract** — sets the nominal radius and re-derives both working radii.

```text
FUNCTION set_switch_distance(distance)
  switch_distance  = distance
  online_distance  = distance * (1 - switch_factor)
  offline_distance = distance * (1 + switch_factor)
```

**Invariants** — this is the hysteresis, and it is expressed as a *fraction of* the
nominal distance rather than as two independent radii. That choice matters: a script or a
console command changing the switch distance (which the game does, for performance) keeps
the hysteresis proportional and cannot accidentally invert the two radii. Two independent
settings could be set the wrong way round and would produce an entity that promotes and
demotes on the same frame.

Promotion tests against the **smaller** radius, demotion against the **larger**. An entity
between them is in neither transition and stays as it is — which is the entire point.

## `set_switch_factor`

**Contract** — changes the hysteresis fraction and re-derives, by re-applying the current
nominal distance. The re-application is necessary because the two working radii are cached
rather than computed on read.

## `online_distance`, `offline_distance`, `switch_distance`

**Contract** — read the three radii. The first two are what the switch decisions compare
against; the third is what a script or the console reads and writes.
