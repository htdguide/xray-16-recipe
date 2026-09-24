# src/xrCore/stdafx.h

> The module's precompiled-header root: the three headers every file in the core begins with.

**Needs** — [`xrCore.h`](xrCore.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../Common/Common.hpp`](../Common/Common.hpp.md) · [`../Common/Util.hpp`](../Common/Util.hpp.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: a build-time artifact.

## Purpose

A compilation-speed device: naming the headers that every file in the module includes lets the compiler process them once. It declares nothing itself.

A rebuild produces nothing corresponding to it. What it *records* is that the core's files all assume the platform-detection layer, the shared utility layer, the whole core umbrella and the scalar extensions are already present — which is worth knowing when reading any file in this directory that appears to use a name it never included.
