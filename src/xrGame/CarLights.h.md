# src/xrGame/CarLights.h

> Declares the vehicle headlight and the group it belongs to, implemented in [`CarLights.cpp`](CarLights.cpp.md).

**Needs** — [`xrEngine/Render.h`](../xrEngine/Render.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CarLights.cpp`](CarLights.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two records that make a vehicle's lights work. Substance in
[`CarLights.cpp`](CarLights.cpp.md).

Exported units:

- `SCarLight` — one lamp: a spot light, a glow sprite and the lens bone, switched together.
  Its operations are create-from-a-configuration-section, switch, and follow-the-bone.
- `CCarLights` — the group the vehicle owns: build the list from the model's `lights`
  section, drive every lamp per frame, switch them all at once, and find one by bone.
- `LIGHTS_STORAGE` — the list type.

The declaration also carries a commented-out block naming six light *roles* — near beam,
far beam, indicators, brake lights, sidelights, door lights — each as an index pair into
the same list. None of it is implemented and no shipped model supplies the data. It is
worth reading as a statement of what a more complete vehicle lighting model would be
shaped like, and worth nothing else.
