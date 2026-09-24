# src/xrEngine/pure_relcase.h

> Declares the relcase-subscriber base; the substance is in [`pure_relcase.cpp`](pure_relcase.cpp.md).

**Needs** — [`pure_relcase.cpp`](pure_relcase.cpp.md) · [`IGame_Level.h`](IGame_Level.h.md)
**Used by** — [`Feel_Touch.h`](Feel_Touch.h.md) · [`Feel_Vision.h`](Feel_Vision.h.md) · [`pure_relcase.cpp`](pure_relcase.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`pure_relcase.cpp`](pure_relcase.cpp.md).

Exported units:

- **`pure_relcase`** — inherit from it, hand it a callback, and the callback fires once for
  every object the level destroys. Holds one field: this subscriber's slot in the level's
  callback table.
