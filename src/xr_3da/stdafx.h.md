# src/xr_3da/stdafx.h

> Names the three layers the executable is allowed to see — the platform vocabulary, the core, and the engine — and nothing else.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`xrEngine/Engine.h`](../xrEngine/Engine.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`stdafx.cpp`](stdafx.cpp.md)

**Tier floor** — T4: it is a list of three names.

## Purpose

In the original this is a precompiled-header stub, which is an incidental build-speed
artifact and does not survive as itself. What survives is the *statement of scope*: the
executable sees the platform vocabulary, the core, and the engine — and does not see the
renderer internals, the game internals, or anything below them. The renderer and game are
reached only through the two narrow module interfaces named in
[`entry_point.cpp`](entry_point.cpp.md).

A rebuilder writing the composition root should hold to that scope deliberately. The
moment the executable can see inside a module, it starts making decisions that belong to
that module, and the composition root stops being readable.

## Exported units

None. It declares nothing of its own.
