# src/xrGame/BlackDrops.h

> Declares an artefact class that adds nothing to the base artefact.

**Needs** — [`Artefact.h`](Artefact.h.md)
**Used by** — [`BlackDrops.cpp`](BlackDrops.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBlackDrops`. See [`BlackDrops.cpp`](BlackDrops.cpp.md): the class exists only to
occupy a class-identifier slot and to be distinguishable by type from script.

Exported units:

- `CBlackDrops` — a base artefact under its own class identifier.
- `Load` — pure delegation.
