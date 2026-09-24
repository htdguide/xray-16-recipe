# src/xrPhysics/stdafx.cpp

> The translation unit that materializes the precompiled header.

**Needs** — [`StdAfx.h`](StdAfx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T1: a build artifact with no runtime meaning.

## Purpose

Entirely incidental: it exists so the toolchain has one compilation unit from which to build the
shared precompiled header. It contributes no code, no state and no decisions, and has no
counterpart in a rebuild.
