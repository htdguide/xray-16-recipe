# src/xrCore/stdafx.cpp

> The translation unit that materializes the precompiled header.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: a build-time artifact.

## Purpose

One line, existing only so the build has something to compile when producing the precompiled header. It contains no code and decides nothing.

A rebuild produces nothing corresponding to it.
