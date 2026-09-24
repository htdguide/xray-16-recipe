# src/xrPhysics/StdAfx.h

> The set of engine modules the physics module compiles against.

**Needs** — [`xrPhysics.h`](xrPhysics.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T1: a compilation-speed device, not a design element.

## Purpose

A precompiled-header aggregate. It carries no decisions of its own, but it does record, in one
place, what the physics module is allowed to reach for: the core (containers, logging, math), the
engine's object and device interfaces, the static collision database, the sound system, and the
material library.

That list is the useful residue. The physics module talks to exactly four things outside itself —
the collision database it queries for triangles, the material table it reads friction and bounce
from, the engine's object interface it writes transforms back through, and the sound system it
reports impacts to. A rebuild that keeps those four edges and nothing more has the dependency shape
right.

The file itself is incidental and disappears in any language with a module system.
