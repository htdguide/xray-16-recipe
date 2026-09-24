# src/xrParticles/stdafx.h

> The module's compilation prelude: the project-wide common prelude, the core runtime, the
> engine's shared defines, and this module's own public header.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`xrEngine/defines.h`](../xrEngine/defines.h.md) · [`psystem.h`](psystem.h.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: a build-system artifact with no runtime meaning.

## Purpose

A precompiled-header root. A rebuild has no equivalent and deletes it.

One thing on this page is not incidental: this prelude is where the particle module acquires
its dependency on the *engine*. The simulation itself needs nothing from the engine — it
needs vectors, a matrix, a random source and a byte reader — but the prelude pulls in the
engine's shared defines, and through them the global flag that says which of the three games'
data is loaded. That flag is read at exactly one place
([`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md)) to decide
whether a two-field tail exists in an authored action record. A rebuild should pass that
choice in as a parameter of the loader rather than reach for a global, which also removes
the backwards build edge described in [`README.md`](README.md).
