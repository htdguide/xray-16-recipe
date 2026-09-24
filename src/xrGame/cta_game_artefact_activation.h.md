# src/xrGame/cta_game_artefact_activation.h

> Declares the capture-the-artefact activation sequence, implemented in [`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md).

**Needs** — [`artefact_activation.h`](artefact_activation.h.md)
**Used by** — [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md) · [`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the activation subclass used by the capture-the-artefact objective. Substance is in
[`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md).

Exported units:

- `CtaArtefactActivation` — the activation. Adds no state.
- `UpdateActivation` — the timeline advance, with the end-of-sequence destruction removed so
  the objective artefact survives being used.
- `ChangeEffects` — overridden to nothing; the mode has no per-stage art.
- `Load` / `Start` / `Stop` / `UpdateEffects` / `SpawnAnomaly` / `PhDataUpdate` — pure
  delegation, present only to declare the surface.
