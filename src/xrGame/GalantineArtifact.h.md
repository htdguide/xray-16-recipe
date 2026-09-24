# src/xrGame/GalantineArtifact.h

> Declares the behaviourless artefact leaf implemented in [`GalantineArtifact.cpp`](GalantineArtifact.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`GalantineArtifact.cpp`](GalantineArtifact.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGalantineArtefact` as an artefact subclass so the class-identifier factory can
name it. Substance — none beyond delegation — is in
[`GalantineArtifact.cpp`](GalantineArtifact.cpp.md).

Exported units:

- `CGalantineArtefact` — an artefact whose only distinguishing feature is its class
  identifier and the configuration section that feeds it.
