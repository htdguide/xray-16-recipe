# src/xrGame/artefact_activation.h

> Declares the artefact-activation sequence implemented in [`artefact_activation.cpp`](artefact_activation.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`Artefact.cpp`](Artefact.cpp.md) · [`artefact_activation.cpp`](artefact_activation.cpp.md) · [`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md) · [`cta_game_artefact_activation.h`](cta_game_artefact_activation.h.md)
**Tier floor** — T3: a declaration and two small records

## Purpose

Declares the surface implemented in
[`artefact_activation.cpp`](artefact_activation.cpp.md). Every method is overridable,
because [`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md) replaces
the spawning step for one multiplayer mode.

Exported units:

- **`EActivationStates`** — the fixed sequence `none, starting, flying, before_spawn,
  spawn_zone`, plus a count. The order is the sequence; the count sizes the state table.
- **`SStateDef`** — one state's presentation row: duration, sound, light colour and range,
  particle and animation, with its own loader from a comma-separated configuration row.
- **`SArtefactActivation`** — the sequence itself, with **`Start`** / **`Stop`**,
  **`UpdateActivation`** (per frame), **`PhDataUpdate`** (per physics step),
  **`ChangeEffects`** / **`UpdateEffects`**, **`SpawnAnomaly`**, **`Load`** and
  **`IsInProgress`**.
