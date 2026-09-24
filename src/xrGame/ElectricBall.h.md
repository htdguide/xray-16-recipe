# src/xrGame/ElectricBall.h

> Declares the carrier-pinned artefact implemented in [`ElectricBall.cpp`](ElectricBall.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`ElectricBall.cpp`](ElectricBall.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares one artefact class overriding its configuration load and the per-frame subclass
update hook. Substance in [`ElectricBall.cpp`](ElectricBall.cpp.md).

Exported units:

- `CElectricBall` — an artefact whose transform tracks its carrier's.
- `UpdateCLChild` — the per-frame hook that does the tracking.
