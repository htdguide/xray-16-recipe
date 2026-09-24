# src/xrGame/GraviArtifact.h

> Declares the hovering artefact implemented in [`GraviArtifact.cpp`](GraviArtifact.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) · [`GraviArtifact.cpp`](GraviArtifact.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGraviArtefact`, the artefact that hovers above the ground. Substance is in
[`GraviArtifact.cpp`](GraviArtifact.cpp.md).

Exported units:

- `CGraviArtefact` — an artefact carrying a hover distance and an unused energy value.
- `Load` — reads the optional hover distance from the section.
- `UpdateCLChild` — the per-frame hover impulse, or the carried-on-the-back transform.
