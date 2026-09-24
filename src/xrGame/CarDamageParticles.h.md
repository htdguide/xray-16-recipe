# src/xrGame/CarDamageParticles.h

> Declares the record holding a vehicle's damage-smoke effect names and emitter bones, implemented in [`CarDamageParticles.cpp`](CarDamageParticles.cpp.md).

**Needs** — _(none beyond the vehicle it belongs to)_
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CarDamageParticles.cpp`](CarDamageParticles.cpp.md) · [`CarWheels.cpp`](CarWheels.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the small record a vehicle keeps for its two levels of visible damage. Substance
in [`CarDamageParticles.cpp`](CarDamageParticles.cpp.md).

Exported units:

- `CCarDamageParticles` — four effect names (body at each of two severities, wheel at each
  of two) and two bone lists.
- `Init`, `Clear` — load from the model's configuration; drop the bone lists.
- `Play1`, `Play2`, `Stop1`, `Stop2` — start and stop a severity's body smoke on every bone
  of that severity.
- `PlayWheel1`, `PlayWheel2` — start a wheel's smoke on one named bone.
