# src/Layers/xrRender/light_gi.h

> The record for one indirect bounce: a virtual light standing in for the light this one reflected off a surface.

**Needs** — [`light_gi.cpp`](light_gi.cpp.md) · [`xrCDB/ISpatial.h`](../../xrCDB/ISpatial.h.md)
**Used by** — [`light.cpp`](light.cpp.md) · [`light.h`](light.h.md) · [`light_gi.cpp`](light_gi.cpp.md)
**Tier floor** — T1: a bare record, sized and laid out because a light may carry hundreds of them and they are iterated per frame.

## Purpose

Declares the one type the indirect-lighting pass works in. The algorithm that produces them is in [`light_gi.cpp`](light_gi.cpp.md).

## State

```text
RECORD IndirectBounce
  position  : vector3   # where the photon hit
  direction : vector3   # the photon's direction reflected about the surface normal
  energy    : real      # normalised so a light's bounces sum to a configured total
  sector    : sector_id # which sector the bounce belongs to
```

Invariants:

- `energy` is meaningful only relative to the other bounces of the *same* light: the set is normalised as a whole, so a single bounce's value says nothing on its own.
- `sector` decides whether a bounce is drawn at all when the visibility pass runs, so a wrong sector makes a bounce leak light through a wall. The value written by the generator is inherited from the parent light rather than detected at the hit point — see [`light_gi.cpp`](light_gi.cpp.md), where this is recorded as a known defect.
