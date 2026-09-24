# src/xrGame/ZudaArtifact.h

> Declares the behaviourless "zuda" artefact leaf implemented in [`ZudaArtifact.cpp`](ZudaArtifact.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`ZudaArtifact.cpp`](ZudaArtifact.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Names `CZudaArtefact` as an artefact subclass so the class-identifier factory can
construct it. Substance — none beyond delegation — is in
[`ZudaArtifact.cpp`](ZudaArtifact.cpp.md).

Exported units:

- `CZudaArtefact` — an artefact whose behaviour is entirely supplied by its configuration
  section.
